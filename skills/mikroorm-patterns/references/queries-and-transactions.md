# Queries and Transactions (MikroORM 7.2)

## Contents
1. Unit of Work: `persist` / `flush` / `remove`
2. Flush modes and the v7 change-detection caveat
3. Finding and populating: `balanced`, `populateHints`, `populateWhere`
4. `Loaded<T, P>` return types
5. QueryBuilder (typed aliases, CTE, union)
6. Kysely (`em.getKysely()`)
7. Raw SQL with named parameters
8. Streaming and batch processing
9. Context isolation: `RequestContext`, `em.fork()`, `@CreateRequestContext`
10. Transactions: `em.transactional`, `@Transactional`, propagation, locking
11. Native PostgreSQL RLS: session context patterns
12. Resource cleanup with `await using`
13. Anti-patterns

---

## 1. Unit of Work

```ts
// ❌ removed in v7
await em.persistAndFlush(user);
await em.removeAndFlush(user);

// ✅ chaining (single entity, one-off)
await em.persist(user).flush();
await em.remove(user).flush();
```

Preferred: one flush per unit of work.

```ts
async importCustomers(rows: CustomerDto[]) {
  for (const row of rows) {
    em.create(Customer, row);           // create() marks for persist already
  }
  await em.flush();                     // ✅ one transaction, batched inserts
}

// ❌ N transactions and N changeset computations
for (const row of rows) {
  await em.persist(em.create(Customer, row)).flush();
}
```

Updating a managed entity needs no `persist`:

```ts
const order = await em.findOneOrFail(Order, id);
order.status = 'paid';
await em.flush(); // dirty-checked against the snapshot → UPDATE orders SET status = ...
```

Detached data (DTO, another fork, cache):

```ts
const managed = em.merge(Order, detachedOrder);            // re-attach
await em.upsert(Customer, { email, name });                // insert or update by unique key
await em.nativeUpdate(Order, { id }, { status: 'void' });  // bypass UoW (no hooks, no identity map update)
```

`em.remove()` on a not-loaded entity: use `em.getReference(Order, id)` then `em.remove(ref)`, or `em.nativeDelete(Order, { id })` if you don't need cascades/hooks.

---

## 2. Flush modes and change detection

| Mode | Behavior |
|---|---|
| `FlushMode.AUTO` (default) | Before a query, flushes only if pending changes overlap the queried entity |
| `FlushMode.COMMIT` | Flush only on commit / explicit `flush()` |
| `FlushMode.ALWAYS` | Flush before every query |

v7 caveat: with `AUTO`, **scalar edits on managed entities are not detected for auto-flush**. `em.flush()` still finds them.

```ts
const book = await em.findOneOrFail(Book, 1);
book.price = 1000;

// ❌ may not see price=1000: auto-flush didn't notice the scalar edit
const expensive = await em.find(Book, { price: { $gte: 1000 } });

// ✅ mark it so auto-flush fires before the query
em.persist(book);
const expensive2 = await em.find(Book, { price: { $gte: 1000 } });
```

If your code relies on read-your-own-writes within a unit of work, either `em.persist()` after mutation or flush explicitly before querying. Set `flushMode` per fork if a workflow needs different semantics: `em.fork({ flushMode: FlushMode.COMMIT })`.

---

## 3. Loading strategies

`balanced` (v7 default for SQL): to-one → JOIN, to-many → separate `select-in` query. This avoids Cartesian products from multiple collections while keeping to-one loads in one round-trip.

```ts
const orders = await em.find(
  Order,
  { status: 'open' },
  {
    populate: ['customer', 'lines', 'lines.product'],
    // customer, lines.product: joined; lines: select-in
  },
);
```

Override per path with `populateHints` (keys are populate paths, with autocomplete):

```ts
const orders = await em.find(Order, { status: 'open' }, {
  populate: ['customer', 'assignee', 'lines', 'auditLog'],
  populateHints: {
    customer: { joinType: 'inner join' },        // drop orders without a customer
    assignee: { joinType: 'left join' },         // optional relation
    auditLog: { strategy: 'select-in' },         // to-one with wide rows: keep it out of the main query
    lines: { limit: 3, orderBy: { createdAt: 'desc' } }, // top-3 per parent, forces select-in, readonly collection
  },
});
```

Other knobs:
- Per query: `strategy: 'select-in' | 'joined' | 'balanced'`.
- Per relation (always wins): `@OneToMany({ ..., strategy: 'select-in' })`.
- Global: `loadStrategy` in config (leave at `balanced`).
- `populateWhere: 'infer'` reuses the main `where` for populated relations (default `'all'` = only filters root).
- `populateOrderBy: { lines: { createdAt: 'desc' } }`.
- Hints for paths not in `populate` are silently ignored — keep them in sync.

N+1 checklist:
1. Any `for` loop that touches `x.relation.prop` or `await x.coll.load()` → move the relation into `populate`.
2. GraphQL resolvers → enable dataloader (`{ dataloader: true }` on `Reference.load()`/`Collection.load()`, or globally; install `dataloader` explicitly — it's an optional peer in v7).
3. Turn on query logging in dev (`debug: ['query', 'query-params']`) and count queries per request.

Pagination: `em.findByCursor(Order, { where: {...}, first: 20, after: cursor, orderBy: { createdAt: 'desc', id: 'desc' } })` — note `where` lives in the options object in v7. Always include a unique tiebreaker in `orderBy`.

`$or` across joined relations is often unindexable; use `unionWhere`:

```ts
await em.find(Customer, {}, {
  unionWhere: [{ email: { $ilike: q } }, { orders: { reference: q } }],
  // unionWhereStrategy: 'union-all' (default) | 'union'
});
```

Collection operators (SQL): `$some`, `$none`, `$every`, `$size`:

```ts
await em.find(Customer, { orders: { $some: { status: 'open' } } });
await em.find(Customer, { orders: { $size: { $gte: 5 } } });
```

---

## 4. `Loaded<T, P>`

```ts
import type { Loaded } from '@mikro-orm/core';

type OrderWithLines = Loaded<Order, 'customer' | 'lines' | 'lines.product'>;

async getOrder(id: string): Promise<OrderWithLines> {
  return this.em.findOneOrFail(Order, id, { populate: ['customer', 'lines', 'lines.product'] });
}

function render(o: OrderWithLines) {
  o.customer.$.name;                 // Ref<Customer> is loaded → `.$` allowed
  o.lines.getItems()[0].product.sku; // collection is loaded → getItems() type-safe
}
```

- Partial loading: `fields: ['id', 'status', 'customer.name']` → result type narrows accordingly (`Loaded<Order, 'customer', 'id' | 'status' | 'customer.name'>`).
- `exclude: ['notes']` for the inverse.
- `em.populate(entities, ['lines'], { populateHints })` for loading after the fact, returns `Loaded<>`.
- `wrap(entity).isInitialized()` / `Reference.isInitialized()` for runtime checks — but prefer making it impossible via types.

---

## 5. QueryBuilder

```ts
import { EntityManager } from '@mikro-orm/postgresql';

const rows = await em.createQueryBuilder(Order, 'o')
  .select(['o.id', 'o.status'])
  .leftJoinAndSelect('o.customer', 'c')        // result type now knows c is loaded
  .where({ 'c.tier': 'pro', status: { $in: ['open', 'paid'] } })
  .orderBy({ 'o.createdAt': 'desc' })
  .limit(50)
  .getResult();                                 // ✅ entities; QB is not awaitable
```

- Chain fluently; assigning `const qb = ...` and continuing in separate statements loses alias typing (reassign `qb = qb.leftJoin(...)` if you must).
- `execute('all' | 'get' | 'run')` → raw rows; `getResult()` / `getSingleResult()` → managed entities; `getResultAndCount()` → `[rows, total]`.
- `where` rejects unknown aliases; top-level `$in` must be under a property.
- Raw fragments: `raw('price + ?', [1])`, `sql\`lower(${'email'})\``; raw and string keys cannot be mixed in one `orderBy` object — use the array form `orderBy([{ [raw('...')]: 'desc' }, { id: 'asc' }])`.
- `qb.as(Order, 'totalAmount')` for subquery aliasing (new signature).

CTE:

```ts
const recent = em.createQueryBuilder(Order, 'o')
  .select(['o.customer', sql`sum(o.total)`.as('spent')])
  .where({ createdAt: { $gte: since } })
  .groupBy('o.customer');

const top = await em.createQueryBuilder(Customer)
  .with('recent', recent)
  .select('*')
  .from('recent', 'r')
  .orderBy({ 'r.spent': 'desc' })
  .limit(10)
  .execute();
```

Recursive CTE:

```ts
await em.createQueryBuilder(Category)
  .withRecursive('tree', raw(
    `select id, parent_id from category where id = ?
     union all
     select c.id, c.parent_id from category c join tree t on c.parent_id = t.id`, [rootId]))
  .select('*').from('tree', 't')
  .execute<{ id: number; parent_id: number | null }[]>();
```

Union as subquery:

```ts
const a = em.createQueryBuilder(Employee).select('id').where({ dept: 'eng' });
const b = em.createQueryBuilder(Employee).select('id').where({ manager: { $ne: null } });
await em.find(Employee, { id: { $in: a.unionAll(b) } });
```

JSON array match (PG): `{ tags: { $elemMatch: { name: 'urgent' } } }`.

---

## 6. Kysely

```ts
// ❌ v6
const knex = em.getKnex();
await knex('orders').where('status', 'open');

// ✅ v7
const kysely = em.getKysely({
  tableNamingStrategy: 'entity',   // selectFrom('Order') instead of table names
  columnNamingStrategy: 'property',// property names in/out
  convertValues: true,             // Date/JSON/custom types converted
  processOnCreateHooks: true,      // fill onCreate defaults on insert
  processOnUpdateHooks: true,      // fill onUpdate on update
});

const stats = await kysely
  .selectFrom('Order')
  .select(eb => ['customer', eb.fn.sum<string>('total').as('spent')])
  .where('status', '=', 'paid')
  .groupBy('customer')
  .execute();
```

- `em.getKysely()` without options = raw Kysely with table/column names.
- Types: `defineEntity` entities are inferred automatically. Decorator entities need `[EntityName]?: 'Order'` and must be passed explicitly in `entities`. Otherwise pass a manual `Database` interface: `em.getKysely<Database>()`, or generate types with `npx mikro-orm discovery:export` (emits `entities`, `Database` and a typed `EntityManager` you can inject in Nest services).
- **Transactions**: `getKysely()` uses the EM's current transaction context — queries inside `em.transactional(async em => em.getKysely()...)` roll back with it. Use a fresh `em.fork()` to run outside.
- Kysely results are plain rows, not managed entities. Convert with `em.map(Order, row)` (7.2: `{ disableIdentityMap: true }` to avoid merging into the identity map).
- Legacy knex snippets: `@mikro-orm/knex-compat` provides `raw(knexQb)` as a migration bridge, not a long-term pattern.

---

## 7. Raw SQL with named parameters (7.2)

```ts
await em.execute(
  'select :col: from orders where tenant_id = :tenant and status = :status',
  { col: 'id', tenant: tenantId, status: 'open' },  // :name binds values, :name: binds identifiers
);
```

PG `::casts` are preserved. Prefer this over string interpolation always.

---

## 8. Streaming and batches

```ts
// ❌ loads everything into memory and the identity map
const all = await em.findAll(Order);

// ✅ cursor-based on PostgreSQL
const fork = em.fork();
let n = 0;
for await (const order of fork.stream(Order, { where: { status: 'open' } })) {
  order.status = 'archived';
  if (++n % 500 === 0) {
    await fork.flush();
    fork.clear();              // bound memory
  }
}
await fork.flush();
```

- Raw-row export: `em.createQueryBuilder(Order).select(['id', 'total']).stream()`.
- Bulk updates without hydration are usually better as `em.nativeUpdate` or a QB `update()` with `raw()` values.
- Under RLS with the `'transaction'` strategy, streaming outside a transaction while a session context is staged throws; wrap in `em.transactional()`.

---

## 9. Context isolation

Global `orm.em` must not be used for work: the identity map would leak between requests and grow unbounded (MikroORM throws a validation error by default).

HTTP (NestJS): `MikroOrmModule.forRoot()` registers request-context middleware automatically; inject `EntityManager` and it resolves the request fork.

Outside HTTP:

```ts
import { Injectable } from '@nestjs/common';
import { MikroORM, EntityManager } from '@mikro-orm/postgresql';
import { CreateRequestContext, EnsureRequestContext } from '@mikro-orm/decorators/legacy';
import { Cron } from '@nestjs/schedule';

@Injectable()
export class BillingJob {
  constructor(private readonly orm: MikroORM, private readonly em: EntityManager) {}

  @Cron('0 3 * * *')
  @CreateRequestContext()          // must decorate an async method
  async run() {
    const due = await this.em.find(Invoice, { dueAt: { $lt: new Date() }, paid: false });
    // ...
    await this.em.flush();
  }

  @EnsureRequestContext()          // reuse caller's context if any, otherwise create one
  async helper() { /* ... */ }
}
```

BullMQ: keep `@Process()`/`process()` as a thin method that calls a separate `@CreateRequestContext()` method.

Manual fork:

```ts
const em = this.em.fork();                        // new identity map, no transaction context
const em2 = this.em.fork({ clear: false });       // keep parent's identity map (rare)
const em3 = this.em.fork({ flushMode: FlushMode.COMMIT, schema: 'tenant_42' });
```

Express-style middleware for custom setups: `app.use((req, res, next) => RequestContext.create(orm.em, next))`. With multiple ORM instances, 7.2 lets `RequestContext.create()` take a `contextName => options` callback to configure each fork.

Event handlers (`@nestjs/event-emitter`, domain events after commit): treat as "outside HTTP" — the request context may be gone by the time an async listener runs. Use `@CreateRequestContext()` on the listener.

Tests: create a fork per test (`em = orm.em.fork()`), or wrap each test in a transaction and roll back.

---

## 10. Transactions

```ts
import { TransactionPropagation, IsolationLevel, LockMode } from '@mikro-orm/core';
import { Transactional } from '@mikro-orm/decorators/legacy';

// Functional — default propagation NESTED (savepoint when already in a tx)
await em.transactional(async tx => {
  const acct = await tx.findOneOrFail(Account, id, { lockMode: LockMode.PESSIMISTIC_WRITE });
  acct.balance -= amount;
  tx.create(LedgerEntry, { account: acct, amount: -amount });
  // flushed automatically before commit
}, { isolationLevel: IsolationLevel.READ_COMMITTED });
```

```ts
@Injectable()
export class TransferService {
  constructor(private readonly em: EntityManager) {}

  @Transactional()                      // default REQUIRED: join outer tx, no savepoint
  async transfer(from: string, to: string, amount: number) { /* uses this.em */ }

  @Transactional({ propagation: TransactionPropagation.REQUIRES_NEW })
  async writeAudit(entry: AuditDto) { /* commits even if caller rolls back */ }
}
```

| Propagation | Behavior |
|---|---|
| `NESTED` (em.transactional default) | savepoint if a tx exists, else new tx |
| `REQUIRED` (@Transactional default) | join existing, else new; inner error rolls back all |
| `REQUIRES_NEW` | suspend outer, independent tx |
| `SUPPORTS` / `MANDATORY` / `NOT_SUPPORTED` / `NEVER` | as in Spring |

Rules:
- Inside the callback, use the callback's `em` (`tx`), not the outer one.
- `clear: true` for an empty identity map in the inner context (default `false` — inner contexts see outer managed entities). Parallel transactions need separate forks.
- After `rollback`, managed entities are detached → retry on a new fork.
- Optimistic locking: `@Property({ version: true }) version!: number;` → `OptimisticLockError` on conflict; `em.findOne(X, id, { lockMode: LockMode.OPTIMISTIC, lockVersion })`.
- Pessimistic locks require an open transaction.
- Nested transaction events: check `args.transaction.savepointName` to tell child from root.

Explicit control (rarely needed):

```ts
const fork = em.fork();
await fork.begin();
try { /* ... */ await fork.commit(); } catch (e) { await fork.rollback(); throw e; }
```

---

## 11. Native PostgreSQL RLS (7.2) — runtime

Policies are declared on entities (see `migrations-and-v7-migration.md` §RLS for schema side). At runtime the ORM pushes a **session context** (variables + optional role) to the connection.

### Config

```ts
defineConfig({
  // ...
  sessionContext: 'transaction', // default: after BEGIN → set_config(k, v, true) + SET LOCAL ROLE; pgBouncer-transaction-pooling safe
  // sessionContext: 'connection' → set_config(k, v, false) + SET ROLE on each acquire after RESET ALL; needs session pooling, resolves EM via RequestContext
});
```

With `'transaction'`, operations outside an explicit transaction get an implicit one when a context is set. Hence: keep `implicitTransactions` enabled; `disableTransactions` + session context throws.

### Per-request tenant context in NestJS

Nest runs **middleware before guards**, so `req.user` set by a Passport/JWT guard does not exist yet in a middleware. Pick one of these:

**A. Guard after authentication (recommended).** Keep the default MikroORM request-context middleware and stage the session context on the request fork once the user is known. No transaction has started yet, so `setSessionContext` is allowed.

```ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { EntityManager } from '@mikro-orm/postgresql';

@Injectable()
export class TenantContextGuard implements CanActivate {
  constructor(private readonly em: EntityManager) {} // resolves to the request fork

  canActivate(ctx: ExecutionContext): boolean {
    const user = ctx.switchToHttp().getRequest().user;
    if (!user?.tenantId) return false;                 // fail closed
    this.em.setSessionContext({ variables: { 'app.tenant': user.tenantId }, role: 'app_user' });
    // using the filter bridge as well? this.em.setFilterParams('byTenant', { tenant: user.tenantId });
    return true;
  }
}
// register after the auth guard: APP_GUARD order, or @UseGuards(JwtAuthGuard, TenantContextGuard)
```

**B. Middleware that resolves the tenant itself** (verifies the JWT or looks up the subdomain directly), with `registerRequestContext: false` so there is exactly one request context per request:

```ts
@Injectable()
export class TenantContextMiddleware implements NestMiddleware {
  constructor(private readonly orm: MikroORM, private readonly tenants: TenantResolver) {}

  async use(req: Request, _res: Response, next: () => void) {
    const tenantId = await this.tenants.fromRequest(req);   // JWT claim / subdomain / header (verified!)
    const em = this.orm.em.fork(tenantId
      ? { session: { variables: { 'app.tenant': tenantId }, role: 'app_user' } }
      : {});
    RequestContext.create(em, next);
  }
}
// MikroOrmModule.forRoot({ ...config, registerRequestContext: false }) and apply this middleware for all routes
```

Never trust an unverified `x-tenant-id` header. Wrapping `next.handle()` inside `RequestContext.create()` in an interceptor is unreliable, because the RxJS subscription can run outside the async context.

### Adjusting context on an existing EM

```ts
em.setSessionContext({ variables: { 'app.tenant': tenantId } }); // merges
em.setSessionContext({ role: 'app_user' });
em.getSessionContext();
em.clearSessionContext();                                         // drops all
```

- Forks inherit the parent's context; passing `session` to `fork()` replaces it.
- Cannot change during an active transaction (throws).
- Values are sent as text; `Date` → ISO 8601.
- Session context is part of the result-cache key → tenants never share cached results.

### Jobs and queues

```ts
@CreateRequestContext()
async processTenant(tenantId: string) {
  const em = this.em.fork({ session: { variables: { 'app.tenant': tenantId }, role: 'app_user' } });
  await em.transactional(async tx => {
    for await (const inv of tx.stream(Invoice, { where: { paid: false } })) { /* ... */ }
  });
}
```

### Filter bridge (app-level WHERE + DB policy from one declaration)

```ts
@Entity()
@Filter({ name: 'byTenant', cond: args => ({ tenantId: args.tenant }), rls: true })
export class Order { /* ... */ }

em.setFilterParams('byTenant', { tenant: tenantId }); // stages where clause AND session var `mikro.byTenant.tenant`
await em.find(Order, {}, { filters: ['byTenant'] });
```

- Override variable name for single-arg filters: `rls: { setting: 'app.tenant' }`.
- Filter conditions must be statically analyzable (no `em`, async, or find-options use) or schema build throws.
- Unset variable → error (fail-closed, intended).

### Errors

Write rejected by `WITH CHECK` → `RowLevelSecurityViolationException` (subclass of `ConstraintViolationException`, SQLSTATE 42501). Map it to 403/404 in a Nest exception filter.

### Bypass for admin/maintenance

Use a separate ORM instance / `contextName` connecting as a role with `BYPASSRLS` (or the table owner without `'force'`), never by clearing the session context on the app connection.

---

## 12. `await using` (7.2)

```ts
// scripts/backfill.ts
import { MikroORM } from '@mikro-orm/postgresql';
import config from '../mikro-orm.config';

await using orm = await MikroORM.init(config); // orm.close() runs at scope exit, even on throw
const em = orm.em.fork();
// ...
```

Requires TS 5.2+ `using` support (`lib: ["esnext.disposable"]` or `ES2024+`), which v7's TS 5.8+ requirement covers. In Nest apps rely on `enableShutdownHooks()` instead.

---

## 13. Anti-patterns

| Anti-pattern | Why it hurts | Do instead |
|---|---|---|
| `persistAndFlush` / `removeAndFlush` | removed | `persist().flush()` or single `flush()` |
| `await em.flush()` inside a loop | N transactions | flush once / per chunk |
| `em.getKnex()` | removed | `em.getKysely()` / QB |
| `await qb` | QB not thenable | `qb.getResult()` / `execute()` |
| `orm.em.find(...)` in cron | global context | `@CreateRequestContext()` / `em.fork()` |
| Sync method with `@CreateRequestContext` | fails validation | make it `async` |
| Assuming `@Transactional` creates savepoints | default is REQUIRED | `em.transactional` or `propagation: NESTED` |
| `loadStrategy: 'joined'` globally | Cartesian blow-up | keep `balanced`, use `populateHints` |
| Touching relations not in `populate` | N+1 / runtime errors | `Loaded<>` return types |
| `findAll()` for exports | memory + identity map | `em.stream()` |
| `SELECT set_config(...)` by hand for tenants | leaks across pooled connections | `em.fork({ session })` / `setSessionContext` |
| Reading `req.user` in a Nest middleware | guards run after middleware, so it's undefined | tenant guard (§11 A) or self-resolving middleware (§11 B) |
| Testing RLS as superuser | bypasses policies | non-owner role, pglite non-superuser role |
