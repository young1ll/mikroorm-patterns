---
name: mikroorm-patterns
description: MikroORM v7 (7.2+) patterns for NestJS + PostgreSQL. Use whenever writing, reviewing or refactoring MikroORM code - entities (@mikro-orm/decorators/legacy or defineEntity), EntityManager/repository queries, flush and Unit of Work, transactions (em.transactional, @Transactional, em.fork, RequestContext), QueryBuilder or em.getKysely(), N+1 and loading strategy (balanced, populateHints, Loaded), PostgreSQL row level security, migrations, v5/v6 upgrades. Also covers DDD aggregates and value objects (embeddables), STI/TPT and polymorphic relations, CQRS read models (virtual/view/materialized entities, projections, DTO serialization), subscribers, audit, transactional outbox, soft delete, multi-tenancy (filters, RLS, schema- or database-per-tenant), bulk upserts, cache, read replicas. Trigger even for a vague mention of "the ORM", a *.entity.ts file, mikro-orm.config.ts or migrations, and especially for code using persistAndFlush, getKnex, @mikro-orm/core decorators or orm.getMigrator().
---

# MikroORM v7 Patterns (NestJS + PostgreSQL)

Target: **MikroORM 7.2.x** (written against 7.2.11), `@mikro-orm/postgresql`, `@mikro-orm/nestjs`, Node 22.17+, TypeScript 5.8+.

Most MikroORM code in the wild (and in model training data) is v5/v6. v7 removed Knex, moved decorators to a separate package, removed several convenience APIs, and changed defaults that silently affect data (UTC timestamps, FK rules). Treat any pattern you "remember" as suspect until it matches the rules below.

## How to use this skill

1. Scan the task against the **Hard rules** — these are the mistakes that compile-fail or corrupt data in v7.
2. Pick the matching **Workflow** section and follow its checklist.
3. Open the reference file only for the area you are working in:

| Working on | Read |
|---|---|
| Entities, relations, `Ref`/`LazyRef`, FK rules, indexes, custom repositories, `defineEntity` | `references/entities-and-relations.md` |
| `persist`/`flush`, find/populate, `Loaded`, QueryBuilder, Kysely, streaming, transactions, `em.fork`, RLS session context | `references/queries-and-transactions.md` |
| v6 → v7 upgrade, config changes, migrations, schema generator, RLS policy migrations | `references/migrations-and-v7-migration.md` |
| Aggregates, value objects/embeddables, custom types, STI/TPT, polymorphic relations, virtual/view/materialized entities, projections, serialization & DTOs, CQRS handlers | `references/domain-and-read-models.md` |
| Hooks & subscribers, audit, outbox & domain events, soft delete, tenancy strategy (filter/RLS/schema/DB), bulk writes, cache, replicas, cancellation, triggers/partitioning, testing | `references/events-tenancy-and-ops.md` |
| An error message or stack trace, surprising behavior, or "review this MikroORM code" | `references/troubleshooting-and-review.md` |

Several tasks span two files (e.g. "multi-tenant outbox" → events + queries/RLS). Read both rather than guessing.

---

## Hard rules (v7.2)

Each rule has a reason. If you deviate, you should be able to say why the reason does not apply.

### Packages & imports

- **Decorators come from `@mikro-orm/decorators/legacy`** (NestJS uses `experimentalDecorators`). `@Entity`, `@Property`, `@ManyToOne`, `@Embeddable`, `@Filter`, lifecycle hooks (`@BeforeCreate` and the rest), `@Transactional`, `@CreateRequestContext` and `@EnsureRequestContext` all live there. ES-spec decorators come from `@mikro-orm/decorators/es`. Importing decorators from `@mikro-orm/core` or the driver package is the v6 pattern: some docs pages and re-exports still show it, but the upgrade guide documents `/legacy`, so use that path everywhere.
- **Set `metadataProvider: ReflectMetadataProvider`** (from `@mikro-orm/decorators/legacy`) and install `reflect-metadata` when using legacy decorators. It is no longer the default, so without it type inference from TS metadata silently stops working.
- **Import `EntityManager`, `EntityRepository`, `QueryBuilder` from `@mikro-orm/postgresql`** (or `@mikro-orm/sql`), not from `@mikro-orm/core`. The core versions lack `createQueryBuilder`, `getKysely`, `execute`.
- **`@mikro-orm/knex` is gone** → `@mikro-orm/sql`. `@mikro-orm/better-sqlite` is gone.

### Persistence

- **No `persistAndFlush` / `removeAndFlush`** — removed in v7. Use `em.persist(x).flush()` / `em.remove(x).flush()`, or better, `em.persist(x)` now and a single `em.flush()` at the end of the unit of work.
- **Never `flush()` inside a loop.** Persist in the loop, flush once. Each flush is a transaction + changeset computation.
- **`em.create()` already marks the entity for persistence** (`persistOnCreate` default). Don't add `em.persist()` after it unless you disabled that option.
- **Managed entities are still dirty-checked on `em.flush()`.** What changed in v7: with `FlushMode.AUTO`, scalar edits on managed entities no longer trigger the *auto-flush before a query*. If you mutate an entity and then query something that must see that change before your explicit flush, call `em.persist(entity)` after mutating (or flush first). `trackChanges` option is removed.
- **Detached entities** (from another fork, a cache, a DTO) are not tracked: re-attach with `em.merge()` or write via `em.upsert()` / `em.nativeUpdate()`. Do not mutate and expect a flush to pick them up.

### Querying

- **No Knex.** `em.getKnex()`, `qb.getKnexQuery()`, `knex.raw()` do not exist. Use, in order of preference:
  1. `em.find` / `em.findOne` / repository methods with typed `FilterQuery`.
  2. `em.createQueryBuilder(Entity, 'alias')` (or `em.qb(Entity)`) — typed aliases, CTEs (`with`, `withRecursive`), `union`/`unionAll`.
  3. `em.getKysely()` for SQL that the ORM cannot express. It runs inside the EM's current transaction.
  4. `em.execute(sql, params)` with named params (`:name`, `:ident:`) as a last resort.
- **QueryBuilder is not awaitable.** Call `.execute()`, `.getResult()`, `.getSingleResult()`, or `.getResultAndCount()`. Build fluently (chain) so alias types accumulate.
- **Top-level operators must sit under a property**: `{ id: { $in: ids } }`, not `{ $in: ... }`.
- **Default load strategy is `balanced`**: to-one relations are JOINed, to-many relations get a separate `select ... where fk in (...)` query. Do not switch to `joined` globally — it reintroduces Cartesian explosion on collections. Override per path with `populateHints`.
- **Return `Loaded<T, 'a' | 'b.c'>`** from service/repository methods that populate, so callers cannot touch an unloaded relation.
- **Large scans use `em.stream()`** (async iterator, cursor-based on PG) — never `findAll()` + loop. Under RLS `'transaction'` strategy, wrap streams in `em.transactional()`.

### Context & transactions

- **Never use the global `orm.em` for work** (it throws by default, unless `allowGlobalContext` is set; don't set it). Inside HTTP requests, `@mikro-orm/nestjs` middleware creates a request context automatically. Nest guards run *after* that middleware, so set tenant/session context in a guard, not in a middleware that reads `req.user`. Outside HTTP (cron, BullMQ/queue consumers, event handlers, bootstrap scripts, tests) use `@CreateRequestContext()` on an **async** top-level method, or an explicit `const em = this.em.fork()`.
- **`@Transactional()` / `@CreateRequestContext()` need an EM source on the instance**: inject `MikroORM`, `EntityManager` or a repository, or pass a callback (`@CreateRequestContext(() => DI.orm)`).
- **`@Transactional()` defaults to `REQUIRED`** (joins the outer transaction, no savepoint — an inner error rolls back everything). **`em.transactional()` defaults to `NESTED`** (savepoint if already in a transaction). Choose explicitly when nesting matters.
- **After rollback, managed entities are detached.** Start the retry on a fresh fork.
- **Scripts/tests: `await using orm = await MikroORM.init(config)`** (7.2+, `Symbol.asyncDispose`) so the pool always closes.

### Schema & data safety

- **FK rules are no longer inferred from `cascade`.** `cascade: [Cascade.REMOVE]` is ORM-level only. Declare DB rules explicitly with `deleteRule` / `updateRule`, or set `schemaGenerator.defaultDeleteRule` / `defaultUpdateRule`.
- **`cascade` no longer infers nullability.** Set `nullable: true` explicitly.
- **`forceUtcTimezone` defaults to `true`.** On a v6 database that stored local time in `timestamp` (without time zone) columns, upgrading silently shifts reads. Decide and set it explicitly; prefer `timestamptz` columns.
- **Use getters**: `orm.schema`, `orm.migrator`, `orm.seeder`, `orm.entityGenerator`. `getSchemaGenerator()`/`getMigrator()` are removed, and methods are shortened (`createSchema` → `create`, `updateSchema` → `update`, `createMigration` → `create`).
- **`driverOptions` is flat** — pass `pg.Pool` options directly (`driverOptions: { ssl: {...} }`), no `connection: {}` wrapper.
- **Entity globs use Node's native glob — no brace expansion.** Use an array (`['dist/a/**/*.entity.js', 'dist/b/**/*.entity.js']`) or explicit entity classes. (Migration/seeder paths still accept braces.)
- **Migrations are reviewed SQL, not `schema:update`.** Never run `schema:update` against shared/prod databases.

### Row level security (7.2+, PostgreSQL only)

- **Declare policies on the entity** (`policies: [...]`, `rowLevelSecurity: true | 'force' | false`) so the schema generator and migrations own them. Hand-written policies outside metadata will be proposed for drop by diffs — adopt them into metadata or set `schemaGenerator.ignorePolicies: true`.
- **Push tenant context through the ORM**, not `SET` statements: `em.fork({ session: { variables: { 'app.tenant': tenantId }, role: 'app_user' } })` or `em.setSessionContext(...)` before any transaction starts.
- **The app must connect as a non-owner role** (or use `rowLevelSecurity: 'force'`). Table owners and superusers bypass RLS; a test that passes as `postgres` proves nothing.
- **Keep `implicitTransactions` enabled** with the default `'transaction'` strategy — setting a session context with it disabled throws (fail-closed by design).

### Application patterns

- **Aggregates:** collections only inside an aggregate (`orphanRemoval`, child `deleteRule: 'cascade'`); other aggregates by `Ref`/id. Repositories only for roots, and they never flush. Put `version: true` on the root and touch the root when children change.
- **Hydration skips constructors.** Constructor invariants protect only `new` objects. Use `new Root(...)` + `persist` for invariant-heavy aggregates and `em.create()` for plain data.
- **Value objects:** embeddables for multi-field VOs (replace, don't mutate), custom `Type` for single-column VOs.
- **Read side doesn't hydrate aggregates.** Use projections (Kysely/QB `execute`), virtual or view entities, or `fields` + `disableIdentityMap`. Never return managed entities from controllers; map them with `serialize()` or DTOs.
- **No `flush()` / `persist()` inside lifecycle hooks.** In `onFlush`, add entities with `uow.computeChangeSet()`; in `beforeFlush` you may persist.
- **Never publish to a broker inside the business transaction.** Write outbox rows in the same flush as the state change, and let a relay publish with `LockMode.PESSIMISTIC_PARTIAL_WRITE` (SKIP LOCKED). Delivery is at-least-once, so consumers dedupe by message id.
- **Filters cover `find*`, `count`, `nativeUpdate` and `nativeDelete` only.** They don't cover inserts, Kysely or raw SQL, and QB queries need `qb.applyFilters()`. RLS is the only tenant guard that covers every path.
- **`nativeUpdate` / `nativeDelete` / bulk QB writes bypass hooks, subscribers, version checks and the identity map.** Refresh or clear the fork afterwards.

---

## Workflows

### A. Adding or changing an entity

1. Choose definition style (see reference): legacy decorators for existing NestJS codebases; `defineEntity` + `setClass` for new modules or when you want inferred Kysely types and no `reflect-metadata`.
2. For every relation decide: owner side, `Ref<T>` / `LazyRef<T>` / plain, `nullable`, `deleteRule`, `updateRule`, and whether the inverse side is needed.
3. Initialize collections: `items = new Collection<Item>(this)`.
4. Add indexes for every FK used in filters and every RLS tenant column.
5. Generate a migration (`npx mikro-orm migration:create --name <x>`), **read the SQL**, then `migration:up` locally. See workflow D.

### B. Writing a service/repository method

1. Inject `EntityManager` from `@mikro-orm/postgresql` or a custom repository.
2. Load with explicit `populate` and, if needed, `populateHints`; type the return as `Loaded<...>`.
3. Mutate entities in memory; `em.create()` new ones; `em.remove()` deleted ones.
4. Finish with one `await em.flush()` (or let `em.transactional`/`@Transactional` flush on commit).
5. Need atomic multi-step logic or locking? Wrap in `em.transactional(async em => ...)` and use the callback's `em`.

### C. Background job / cron / queue consumer / event handler

1. Put `@CreateRequestContext()` on the async entry method (class must have `MikroORM` or `EntityManager` injected), or create `const em = this.em.fork()` manually.
2. For multi-tenant jobs, fork per tenant with `session` variables (RLS) and/or filter params.
3. Batch: `for await (const row of em.stream(...))`, persist in chunks, `flush()` + `em.clear()` per chunk to bound identity-map memory.

### D. Migration

1. `migration:create` (diff against snapshot) → open the generated file → review every statement (drops, type changes, policy drop/create, FK rules).
2. Add data backfills with `this.addSql(...)` / `this.execute(...)` in the same migration when a column becomes NOT NULL.
3. `migration:up` on a disposable DB; `migration:check` must report no diff afterwards.
4. Commit migration **and** snapshot together. In CI/prod run `migration:up` from compiled `dist/migrations`.

### E. Command (write) vs query (read) handler

1. Command: load the aggregate with its full child graph → call domain methods → `em.create(OutboxMessage, ...)` for recorded events → one flush, in `@Transactional()` or `em.transactional()`.
2. Query: use a fresh fork (it may hit a replica), project into a DTO with Kysely/QB or a view entity, never flush, and use `connectionType: 'write'` for read-after-write.
3. Side effects (email, broker) run after commit: outbox relay, or code after the `await em.transactional(...)` returns.

### F. Choosing a tenancy strategy

1. Default for PG SaaS: shared schema + RLS (with the filter bridge if you want app-level predicates too).
2. Regulated or very large tenants: schema-per-tenant (`schema: '*'`, `migrator.up({ schema })` per tenant in a deploy job) or a DB per tenant (an ORM registry with small pools).
3. For every strategy: one fork per request or job, tenant id from the request context, and a cross-tenant isolation test per tenant-scoped entity.

### G. Code review or debugging

1. Debugging: match the symptom in `references/troubleshooting-and-review.md` §1 before guessing. Ask for the logged SQL (`debug: ['query', 'query-params']`) when the cause is unclear.
2. Review: walk the severity-ordered checklist there (§3): tenant isolation and SQL injection first, v7 hygiene last. Report in the §4 format.
3. For every finding, give the fix with v7 import paths, and say whether a migration is needed.

### H. Upgrading v6 code

Grep for these patterns and fix each match (details in `references/migrations-and-v7-migration.md`):

```
persistAndFlush|removeAndFlush|getKnex|getKnexQuery|@mikro-orm/knex
from '@mikro-orm/core'.*\b(Entity|Property|ManyToOne|OneToMany|ManyToMany|OneToOne|PrimaryKey|Transactional|CreateRequestContext)\b
getMigrator\(|getSchemaGenerator\(|getSeeder\(|initSync|ArrayCollection|trackChanges
connection: \{   (inside driverOptions)
await .*createQueryBuilder\(   (awaiting a QB directly)
```

---

## Decision checklists

**Which reference type for a to-one relation?**
- Want zero runtime indirection but compile-time protection → `LazyRef<T>` (7.1+): only PK is accessible until `Loaded<>` narrows it.
- Want explicit `.load()` / `.$` / `.unwrap()` and safe `em.getReference` semantics → `Ref<T>` (`ref: true`).
- Small, always-loaded aggregate internals → plain `T`.
- Circular import between entity files (ESM) → wrap the type in `Rel<T>` (type-only, for `emitDecoratorMetadata` safety).

**Which query tool?**
- Can be expressed as a filter object → `em.find`.
- Needs joins with conditions, aggregates, CTEs, unions, subqueries → QueryBuilder.
- Needs window functions, `lateral`, vendor SQL, bulk `insert ... select` → `em.getKysely()`.
- `$or` across joined relations is slow → `unionWhere` find option (one UNION ALL branch per condition, each uses its own index).

**Which loading strategy?**
- Default `balanced` for almost everything.
- A required to-one that should filter parents → `populateHints: { rel: { joinType: 'inner join' } }`.
- A to-one that is huge/rarely needed in the same row → `populateHints: { rel: { strategy: 'select-in' } }`.
- Only top-N children per parent → `populateHints: { children: { limit: 5, orderBy: {...} } }` (forces select-in, result is readonly).

**Transaction propagation?**
- Service method that must participate in caller's transaction → `@Transactional()` (REQUIRED).
- Inner step whose failure must not abort the outer work → `em.transactional(fn)` (NESTED → savepoint).
- Audit of *attempts* that must survive the caller's rollback → `REQUIRES_NEW`. The outbox is the opposite: it must share the caller's transaction (`REQUIRED`), or events go out for changes that never committed.

**Read model?**
- Screen shows part of one aggregate → `fields` + `populate` (+ `disableIdentityMap`).
- Aggregated report evaluated at query time → virtual entity (`expression`).
- Reusable SQL also read by BI/other services → view entity (`view: true`).
- Expensive aggregate, staleness OK → materialized view + scheduled `refreshMaterializedView`.
- Hot list endpoint → Kysely projection into a DTO interface.

**Hierarchy?**
- Subtypes share most columns → STI. Many distinct columns or constraints → TPT.
- Unrelated targets ("commentable", "attachable") → polymorphic relation. It has no DB FK, so add an index and orphan cleanup.

**Where does cross-cutting logic go?**
- Entity-local derivation (slug, normalization) → lifecycle hook.
- Audit/metrics across entities → `onFlush` subscriber.
- Integration events → explicit outbox in the command handler (preferred) or a `beforeFlush` subscriber.

---

## Before / After cheat sheet

**1. Imports & metadata**
```ts
// ❌ v6
import { Entity, Property, ManyToOne, Transactional } from '@mikro-orm/core';
// ✅ v7
import { Entity, Property, ManyToOne, Transactional, ReflectMetadataProvider } from '@mikro-orm/decorators/legacy';
import { EntityManager, EntityRepository } from '@mikro-orm/postgresql';
```

**2. Persistence**
```ts
// ❌
for (const dto of dtos) await em.persistAndFlush(em.create(Item, dto));
// ✅
for (const dto of dtos) em.create(Item, dto);
await em.flush();
```

**3. Raw SQL & loading**
```ts
// ❌
const rows = await em.getKnex()('orders').where({ status: 'open' });
const orders = await em.find(Order, {}, { populate: ['lines', 'tags'], strategy: 'joined' }); // cartesian
// ✅
const rows = await em.getKysely().selectFrom('orders').selectAll().where('status', '=', 'open').execute();
const orders: Loaded<Order, 'customer' | 'lines'>[] = await em.find(Order, {}, {
  populate: ['customer', 'lines'],                      // balanced: customer JOIN, lines SELECT-IN
  populateHints: { customer: { joinType: 'inner join' } },
});
```

**4. Context & transactions**
```ts
// ❌ cron using the global EM
@Cron('* * * * *') tick() { return this.orm.em.find(Job, {}); }
// ✅
@Cron('* * * * *') @CreateRequestContext() async tick() { return this.em.find(Job, {}); }
```

**5. RLS, schema, migrations**
```ts
// ❌ manual tenant switch on a pooled connection; v6 getters
await em.execute(`select set_config('app.tenant', '${tenantId}', false)`);
await orm.getMigrator().up();
// ✅
const tenantEm = em.fork({ session: { variables: { 'app.tenant': tenantId }, role: 'app_user' } });
await orm.migrator.up();
```

---

## Minimal NestJS config skeleton

```ts
// mikro-orm.config.ts
import { defineConfig, PostgreSqlDriver } from '@mikro-orm/postgresql';
import { ReflectMetadataProvider } from '@mikro-orm/decorators/legacy';
import { Migrator } from '@mikro-orm/migrations';

export default defineConfig({
  driver: PostgreSqlDriver,
  metadataProvider: ReflectMetadataProvider,
  entities: ['dist/**/*.entity.js'],      // no {a,b} braces
  entitiesTs: ['src/**/*.entity.ts'],
  clientUrl: process.env.DATABASE_URL,
  driverOptions: { max: 20 },             // pg.Pool options, flat
  forceUtcTimezone: true,                 // explicit, even though it is the default
  // loadStrategy: 'balanced',            // default since v7
  schemaGenerator: { defaultDeleteRule: 'no action', defaultUpdateRule: 'cascade' },
  migrations: { path: 'dist/migrations', pathTs: 'src/migrations', snapshot: true },
  extensions: [Migrator],
});
```

```ts
// app.module.ts
import { MikroOrmModule } from '@mikro-orm/nestjs';
import config from './mikro-orm.config';

@Module({ imports: [MikroOrmModule.forRoot(config)] }) // config argument is required in v7
export class AppModule {}
```

Call `app.enableShutdownHooks()` in `main.ts` so the pool closes on SIGTERM.

---

## Output expectations

When producing MikroORM code with this skill:
- Show imports explicitly (package paths are the #1 v7 error source).
- Prefer complete, compilable snippets over fragments.
- If you override a default (`loadStrategy`, `forceUtcTimezone`, propagation), add a one-line comment saying why.
- When touching schema, state which migration must be generated and what SQL to expect.
- If the user's code is v6-style, point out each outdated call and give the v7 replacement.
- Flag API surface that is new or less documented (polymorphic relations, triggers, partitioning, routines) and suggest checking the current docs before relying on it.
- Don't invent options. If you are not sure an option exists in 7.2, say so and show the documented alternative.
