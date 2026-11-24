# Events, Multi-Tenancy and Operational Patterns (MikroORM 7.2)

## Contents
1. Lifecycle hooks vs event subscribers
2. Flush events: auditing, derived data
3. Domain events and the transactional outbox
4. Soft delete
5. Multi-tenancy strategy matrix
6. Shared schema: filters, RLS, or both
7. Schema-per-tenant
8. Database-per-tenant
9. Bulk writes: upsertMany, insertFrom, clone
10. Result cache
11. Read replicas
12. Query cancellation (AbortSignal)
13. DB-side features: triggers, partitioning, routines, partial indexes
14. Testing patterns
15. Anti-patterns

---

## 1. Hooks vs subscribers

| | Lifecycle hooks (entity methods) | `EventSubscriber` |
|---|---|---|
| Scope | one entity class | selected entities or all |
| Dependencies | none (no DI) | can be Nest `@Injectable()` |
| Use for | entity-local derivations (slug, normalization) | cross-cutting concerns: audit, outbox, soft delete, metrics |

```ts
import { Entity, Property, BeforeCreate, BeforeUpdate } from '@mikro-orm/decorators/legacy';
import type { EventArgs } from '@mikro-orm/core';

@Entity()
export class Article {
  @Property() title!: string;
  @Property() slug!: string;

  @BeforeCreate() @BeforeUpdate()
  syncSlug(args: EventArgs<Article>) { this.slug = slugify(this.title); }
}
```

`defineEntity`: register hooks with `Schema.addHook('beforeCreate', args => ...)` after defining the class for full typing (inline `hooks` give `args.entity: any`).

Subscriber registered from Nest DI:

```ts
import { Injectable } from '@nestjs/common';
import { EntityManager } from '@mikro-orm/postgresql';
import type { EventSubscriber, FlushEventArgs, TransactionEventArgs } from '@mikro-orm/core';

@Injectable()
export class AuditSubscriber implements EventSubscriber {
  constructor(em: EntityManager, private readonly clock: Clock) {
    em.getEventManager().registerSubscriber(this);
  }
  async onFlush(args: FlushEventArgs) { /* §2 */ }
}
```

(Or config `subscribers: [new X()]` when no DI is needed.)

Hard rules inside hooks/subscribers:
- Never call `em.flush()` (throws) or `em.persist()` (undefined behavior) from entity hooks.
- `onInit` is synchronous and doesn't fire for `new Entity()`.
- `beforeUpdate`/`afterUpdate` fire only for scalar + owning to-one changes. Collection changes → inspect `uow.getCollectionUpdates()` in a flush event.
- `getSubscribedEntities()` doesn't filter flush or transaction events — check `cs.entity instanceof X` yourself.
- Hooks run before subscribers.

---

## 2. Flush events

| Event | Safe to do |
|---|---|
| `beforeFlush` | create and persist new entities (`args.em.persist(...)`) — they join this flush |
| `onFlush` | inspect/modify change sets; add entities via `uow.computeChangeSet(e)`; re-diff via `uow.recomputeSingleChangeSet(e)` |
| `afterFlush` | notifications, cache invalidation (queries are done; transaction may not be committed yet if inside an outer tx) |

Audit log from change sets:

```ts
import { ChangeSetType, type FlushEventArgs } from '@mikro-orm/core';

async onFlush({ uow }: FlushEventArgs) {
  for (const cs of [...uow.getChangeSets()]) {   // copy: we add change sets while iterating
    if (!(cs.entity instanceof Auditable)) continue;
    const log = new AuditLog();                 // `new`, not em.create (no persist inside events)
    log.entity = cs.name;
    log.entityId = String(cs.getPrimaryKey());
    log.action = cs.type;                       // 'create' | 'update' | 'delete' (string enum ChangeSetType)
    log.diff = cs.payload;                      // changed columns only
    log.actorId = this.als.getStore()?.userId;  // from your AsyncLocalStorage request context
    uow.computeChangeSet(log);                  // schedule INSERT in this flush
  }
}
```

Derived field on another entity:

```ts
async onFlush({ uow }: FlushEventArgs) {
  for (const cs of [...uow.getChangeSets()]) {
    if (cs.entity instanceof OrderLine && cs.type === ChangeSetType.CREATE) {
      const order = cs.entity.order;              // Rel<Order> (plain entity), already in the UoW
      order.lineCount += 1;
      // parent may have no change set yet → compute one; if it already has one, use recomputeSingleChangeSet(order)
      uow.computeChangeSet(order);
    }
  }
}
```

Keep flush subscribers cheap and synchronous-ish: they run inside the flush transaction for every flush in the app.

---

## 3. Domain events and the transactional outbox

Goal: state change and "event happened" are committed atomically; publishing to a broker happens after commit, at-least-once.

**Preferred: explicit in the application layer** (no magic, easy to test):

```ts
// domain
export abstract class AggregateRoot {
  #events: DomainEvent[] = [];
  protected record(e: DomainEvent) { this.#events.push(e); }
  pullEvents() { const e = this.#events; this.#events = []; return e; }
}
```

`#events` is a native private field, so MikroORM never treats it as a property.

```ts
@Entity({ tableName: 'outbox' })
@Index({ properties: ['status', 'occurredAt'] })
export class OutboxMessage {
  @PrimaryKey({ type: 'uuid' }) id = crypto.randomUUID();
  @Property() type!: string;
  @Property({ type: 'uuid' }) aggregateId!: string;
  @Property({ type: 'jsonb' }) payload!: unknown;
  @Property({ type: 'timestamptz' }) occurredAt = new Date();
  @Property() status: 'pending' | 'published' | 'dead' = 'pending';
  @Property({ type: 'timestamptz', nullable: true }) publishedAt?: Date | null;
  @Property() attempts = 0;
}
```

```ts
@Transactional()
async placeOrder(cmd: PlaceOrder) {
  const order = await this.orders.load(cmd.orderId);
  order.place();
  for (const e of order.pullEvents()) {
    this.em.create(OutboxMessage, { type: e.type, aggregateId: order.id, payload: e });
  }
  // single flush on commit writes order + outbox rows atomically
}
```

**Alternative: generic `beforeFlush` subscriber** that scans managed `AggregateRoot`s and creates outbox rows — fine for many aggregates, but keep it deterministic (only roots in the identity map with pending events).

Relay (separate worker, competing consumers):

```ts
@Cron('*/5 * * * * *')
@CreateRequestContext()
async relay() {
  await this.em.transactional(async tx => {
    const batch = await tx.find(OutboxMessage, { status: 'pending' }, {
      orderBy: { occurredAt: 'asc' },
      limit: 100,
      lockMode: LockMode.PESSIMISTIC_PARTIAL_WRITE,   // FOR UPDATE SKIP LOCKED → safe with N relays
    });
    for (const msg of batch) {
      try {
        await this.broker.publish(msg.type, msg.payload, { messageId: msg.id }); // consumers dedupe by id
        msg.status = 'published';
        msg.publishedAt = new Date();
      } catch {
        msg.attempts += 1;
        if (msg.attempts >= 10) msg.status = 'dead';   // poison message: alert + manual replay
      }
    }
  }); // one flush on commit marks the whole batch
}
```

- Ordering: `SKIP LOCKED` with several relays gives no global order. If consumers need per-aggregate order, partition relays by `hash(aggregateId)` or let consumers reorder by a per-aggregate sequence number stored in the outbox row.

- In-process side effects that must only happen after commit: run them after `await em.transactional(...)` resolves, or use the `afterTransactionCommit` transaction event (entity-agnostic; fires for nested savepoints too — check `args.transaction.savepointName`).
- Never publish to the broker inside the **business** transaction (the one that changes the aggregate). A rollback would leave a phantom event. The relay publishing while it holds its own `SKIP LOCKED` transaction is fine: if it crashes after publishing, the row stays unpublished and is re-sent (at-least-once), so consumers must dedupe by message id.
- Under RLS, the relay runs as a privileged role or iterates tenants with per-tenant session context.
- Purge/partition the outbox (see `partitionBy` §13).

---

## 4. Soft delete

```ts
@Entity()
@Filter({ name: 'notDeleted', cond: { deletedAt: null }, default: true })
@Unique({ properties: ['tenantId', 'email'], where: { deletedAt: null } }) // 7.1 partial unique index
export class Customer {
  @Property({ type: 'timestamptz', nullable: true }) deletedAt?: Date | null;

  softDelete(now = new Date()) { this.deletedAt = now; }
}
```

- Prefer an explicit domain method (`customer.softDelete()` + flush) over intercepting `em.remove()`. If you must turn `em.remove()` into an update globally, do it in an `onFlush` subscriber with `uow.computeChangeSet(cs.entity, ChangeSetType.UPDATE)` per the events docs — and test it carefully.
- Bypass for admin/restore: `em.find(Customer, where, { filters: { notDeleted: false } })`; `filters: false` disables all filters.
- Relations: filters on M:1/1:1 targets go into the JOIN ON; a NOT NULL FK becomes INNER JOIN and can hide parents pointing at soft-deleted rows. Use `strict`/`autoJoinRefsForFilters` deliberately.
- QueryBuilder needs `qb.applyFilters()`; Kysely ignores filters entirely — add `where deleted_at is null` yourself.
- TPT: filters don't see child rows removed by DB cascade.
- Retention: hard-delete old soft-deleted rows in a scheduled job with `nativeDelete`.

---

## 5. Multi-tenancy strategy matrix

| Strategy | Isolation | Ops cost | MikroORM mechanism | Use when |
|---|---|---|---|---|
| Shared schema + app filter | app-only | lowest | global `@Filter` / `em.addFilter` + `setFilterParams` | internal tools, low risk |
| Shared schema + **RLS** | DB-enforced | low | 7.2 `policies` + `session` context | default for SaaS on PG |
| Shared schema + RLS **filter bridge** | DB + app | low | filter `rls: true` | want query planner hints + DB guarantee |
| Schema-per-tenant | strong, per-schema backup/restore | medium (N migrations) | `schema: '*'` entities, `em.fork({ schema })`, `migrator.up({ schema })` | regulated tenants, tens–hundreds of tenants |
| Database-per-tenant | strongest | high (N pools) | one `MikroORM` instance per tenant | enterprise tenants, data residency |
| Hybrid | per tier | varies | shared RLS for small tenants, dedicated schema/DB for large | mixed customer base |

Common requirements regardless of strategy:
- Tenant id is resolved once (auth guard or a self-verifying middleware) and carried in `AsyncLocalStorage`/request context, never passed by callers into repository signatures ad hoc.
- Background jobs iterate tenants explicitly, one fork per tenant.
- Cache keys include the tenant (RLS session context already does this for the result cache).
- Tests assert cross-tenant isolation, not just happy paths.

---

## 6. Shared schema: filters, RLS, or both

App-level global filter (no RLS):

```ts
// config
filters: {
  tenant: { name: 'tenant', cond: (args) => ({ tenantId: args.id }), default: true, entity: [Order, Customer, Invoice] },
}
// per request
const em = orm.em.fork();
em.setFilterParams('tenant', { id: tenantId });
```

Filters apply to `find*`, `count`, `nativeUpdate` and `nativeDelete` (with `type` = `'read' | 'update' | 'delete'`), but **inserts are not filtered**, and neither are QB queries without `qb.applyFilters()`, Kysely or raw SQL. Set `tenantId` on create (e.g. `@BeforeCreate` from request context or in a base `TenantOwned` constructor) and add `check`-style validation.

RLS (DB-enforced) — see `queries-and-transactions.md` §11 for runtime and `migrations-and-v7-migration.md` §6 for policies. Combining both via `rls: true` on the filter gives one declaration that generates the policy and stages the session variable (`mikro.<filter>.<arg>`).

Decision: if a bug in a single query could leak another tenant's data and that matters, use RLS.

---

## 7. Schema-per-tenant

```ts
@Entity({ schema: '*' })            // wildcard: schema chosen at runtime
export class Invoice { /* ... */ }

@Entity({ schema: 'public' })       // shared catalog
export class Tenant { @PrimaryKey() id!: string; @Property() schemaName!: string; }
```

```ts
// request scope
const em = orm.em.fork({ schema: tenant.schemaName });
RequestContext.create(em, next);

// per-operation override
await em.find(Invoice, {}, { schema: 'tenant_acme' });
// inserts into an explicit schema: em.createQueryBuilder(Invoice).withSchema('tenant_acme').insert(...)
```

Migrations:

```ts
migrations: { includeWildcardSchema: true }   // emit unqualified DDL for wildcard entities
```

```ts
// provisioning a tenant
await orm.em.execute(`create schema if not exists :s:`, { s: schemaName });
await orm.migrator.up({ schema: schemaName });  // 7.1+: each schema keeps its own migration history

// deploy: migrate every tenant schema (job, not app boot)
for (const t of await orm.em.fork().find(Tenant, {})) {
  await orm.migrator.up({ schema: t.schemaName });
}
```

- One connection must have access to all schemas; pool size is shared.
- `SchemaGenerator` skips wildcard entities unless you pass a schema.
- Track per-tenant migration failures; make the deploy job resumable.
- Beyond a few hundred schemas, catalog bloat and migration time dominate — revisit RLS.

---

## 8. Database-per-tenant

`MikroOrmModule` DI binds one ORM per `contextName`, so dynamic tenants need a registry:

```ts
@Injectable()
export class TenantOrmRegistry implements OnApplicationShutdown {
  #orms = new Map<string, Promise<MikroORM>>();

  get(tenant: TenantConfig): Promise<MikroORM> {
    let orm = this.#orms.get(tenant.id);
    if (!orm) {
      orm = MikroORM.init({ ...baseConfig, clientUrl: tenant.dbUrl, driverOptions: { max: 5 } });
      this.#orms.set(tenant.id, orm);
    }
    return orm;
  }

  async onApplicationShutdown() {
    await Promise.all([...this.#orms.values()].map(async o => (await o).close()));
  }
}
```

- Metadata discovery per instance is expensive: share discovered metadata via `metadataCache` / `discovery:export` barrels, and keep pools small (total connections = tenants × max).
- Evict idle tenants (LRU) and `close()` them.
- Migrations: iterate tenant DBs in a deploy job with `orm.migrator.up()` per instance.
- A known small set of databases → static `MikroOrmModule.forRoot({ contextName })` per DB instead.

---

## 9. Bulk writes

```ts
// idempotent sync from an external catalog
await em.upsertMany(Product, rows, {
  onConflictFields: ['sku'],
  onConflictAction: 'merge',
  onConflictExcludeFields: ['createdAt'],
});

// insert ... select (7.1)
await em.qb(ArchivedOrder).insertFrom(
  em.qb(Order, 'o').select(['o.id', 'o.total', 'o.placedAt']).where({ placedAt: { $lt: cutoff } }),
).execute();

// server-side clone with overrides (7.1)
const copy = await em.clone(Quote, { id: quoteId }, { status: 'draft' });

// set-based updates without hydration
await em.nativeUpdate(Order, { status: 'placed', placedAt: { $lt: cutoff } }, { status: 'expired' });
await em.createQueryBuilder(Product).update({ stock: raw('stock - ?', [qty]) }).where({ id }).execute();
```

- `nativeUpdate`/`nativeDelete`/QB updates bypass hooks, subscribers, optimistic locking and the identity map (filters still apply to `nativeUpdate`/`nativeDelete`; QB needs `applyFilters()`). Managed instances in the current fork become stale → `em.clear()` or `em.refresh(x)`.
- Large imports: chunk (500–5000 rows), one flush/upsert per chunk, `em.clear()` between chunks.

---

## 10. Result cache

```ts
// config
resultCache: { adapter: RedisCacheAdapter, expiration: 60_000, options: { client: redis } }, // your CacheAdapter impl

// per query
const plans = await em.find(Plan, { active: true }, { cache: ['plans:active', 300_000] });
await em.clearCache('plans:active');                  // after a write that changes plans
const n = await em.createQueryBuilder(Order).where({ status: 'open' }).cache(10_000).getCount();
```

Custom adapter: implement `CacheAdapter` (`get`, `set`, `remove`, `clear`, optional `close`). Default is in-memory, 1 s, per ORM instance (not shared between replicas).

Cache only reference data or tolerant reads. Cached results are re-hydrated into the identity map; don't mutate and flush cached entities without reloading.

---

## 11. Read replicas

```ts
defineConfig({
  clientUrl: process.env.DB_PRIMARY,
  replicas: [{ host: 'replica-1.internal' }, { host: 'replica-2.internal' }], // only fields that differ
  preferReadReplicas: true, // default
});
```

- Outside transactions, `find`/`count` go to a random replica; inside `em.transactional` everything uses the primary.
- Read-your-writes: `em.findOne(Order, id, { connectionType: 'write' })` or QB third argument `'write'`.
- `em.getConnection('read')` for raw reads.
- Replication lag + outbox/projections: query models fed asynchronously are naturally eventually consistent — document it in the API.

---

## 12. Query cancellation (7.1+)

```ts
@Get('report')
async report(@Req() req: Request) {
  const ctrl = new AbortController();
  req.on('close', () => ctrl.abort());
  const em = this.em.fork({ signal: ctrl.signal });
  return em.find(BigReport, {}, { inflightQueryAbortStrategy: 'cancel query' });
}
```

Strategies: `'ignore query'` (default — stop waiting, query finishes on the server), `'cancel query'` (`pg_cancel_backend`), `'kill session'`. Streams always behave as `'ignore query'`.

---

## 13. DB-side features (schema-managed)

```ts
// Trigger (7.1) — created/diffed/dropped by the schema generator
@Entity()
@Trigger({ name: 'order_touch', timing: 'before', events: ['update'], body: `new.updated_at = now(); return new;` })
export class Order { /* ... */ }

// PostgreSQL partitioning (7.1)
@Entity({ tableName: 'event_log', partitionBy: { type: 'range', expression: ['occurredAt'] } })
export class EventLog { /* ... */ }

// Partial unique index (7.1)
@Unique({ properties: ['email'], where: { deletedAt: null } })

// Stored routine (7.1, experimental)
const HashUser = new Routine({ name: 'hash_user', type: 'function',
  params: { name: { type: 'varchar(255)' } }, returns: { runtimeType: 'string' }, body: 'SELECT ...' });
await em.callRoutine(HashUser, { name: 'jon' });
```

Check the exact option shapes (trigger `body` callback form, partition definitions) in the current docs before generating migrations; always review the emitted SQL.

---

## 14. Testing patterns

- **Real PostgreSQL semantics in-process**: `@mikro-orm/pglite` with `dbName: 'memory://'`; `await orm.schema.refresh()` once per file. (No streaming on pglite; RLS needs a non-superuser role.)
- **Per-test isolation**: fork per test (`em = orm.em.fork()`); for speed, wrap each test in `em.begin()`/`em.rollback()`.
- **Factories**: `@mikro-orm/seeder` `Factory` classes for test data; never share entity instances across forks.
- **Assert query counts** for N+1-sensitive endpoints via a custom logger or `debug` capture.
- **Multi-tenant**: always one cross-tenant read test and one cross-tenant write test per tenant-scoped entity.
- **Outbox**: assert that rolled-back commands leave no outbox rows, and that relays with concurrent workers don't double-publish (SKIP LOCKED).
- **Disposal**: `await using orm = await MikroORM.init(testConfig)` in setup helpers.

---

## 15. Anti-patterns

| Anti-pattern | Fix |
|---|---|
| `em.flush()` / `em.persist()` inside hooks | `uow.computeChangeSet` in `onFlush`, or `beforeFlush` persist |
| Publishing to a broker inside the DB transaction | outbox + relay after commit |
| Relay without `SKIP LOCKED` | `LockMode.PESSIMISTIC_PARTIAL_WRITE` |
| Relying on `beforeUpdate` for collection changes | `uow.getCollectionUpdates()` |
| Soft delete with plain unique constraints | partial unique index `where deleted_at is null` |
| Kysely/raw SQL or QB without `applyFilters()` assuming filters apply | add predicates explicitly (RLS still applies) |
| Tenant id passed manually into every repository method | request-scoped fork with filter params / session context |
| Migrating hundreds of tenant schemas on app boot | dedicated, resumable migration job |
| One unbounded ORM instance per tenant | registry with small pools + LRU eviction |
| `nativeUpdate` then reading stale managed entities | `em.refresh()` / `em.clear()` |
| Caching tenant data with global keys | include tenant in key (automatic under RLS session context) |
| Read-after-write on replicas | `connectionType: 'write'` |
