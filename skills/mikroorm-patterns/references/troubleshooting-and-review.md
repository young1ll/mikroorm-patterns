# Troubleshooting and Code Review (MikroORM 7.2)

Use this file when the user pastes an error or stack trace, when behavior is surprising ("my update isn't saved", "N+1 in prod", "tenant B sees tenant A"), or when asked to review MikroORM code.

## Contents
1. Symptom → cause → fix
2. Debugging toolkit
3. Code review checklist (severity-ordered)
4. Review output format

---

## 1. Symptom → cause → fix

The error texts below are paraphrased. Match on the meaning, not exact wording.

### Startup / discovery / build

| Symptom | Likely cause | Fix |
|---|---|---|
| `SyntaxError` or `Cannot use import statement` / `ERR_REQUIRE_ESM`-style failures after upgrade | v7 is native ESM; TS `moduleResolution: node` breaks `exports` maps | Node 22.17+, TS 5.8+, `module`/`moduleResolution: nodenext` (or `bundler`) |
| Decorators "do nothing", property types wrong / `undefined` type in metadata | no `metadataProvider` (ReflectMetadataProvider no longer default) or `reflect-metadata` not imported first | `metadataProvider: ReflectMetadataProvider` from `@mikro-orm/decorators/legacy`; `import 'reflect-metadata'` at entry |
| `Cannot access 'X' before initialization` / circular import between entity files | `emitDecoratorMetadata` references the class eagerly | type relation as `Rel<X>` (or `Ref<X>`), use `() => X` in decorators |
| Entity "not discovered" / "not part of the metadata" | glob with `{a,b}` braces (native glob), wrong `entities` vs `entitiesTs`, entity only reachable via relation with `autoLoadEntities` | explicit arrays/classes; add to `forFeature()` or `entities` |
| `MikroOrmModule.forRoot()` throws without arguments | v7 requires a config | pass config |
| Env vars ignored / config file values win unexpectedly | v7 precedence: explicit > env > config file | `preferEnvVars: true` or move values |
| CLI can't load TS config | `ts-node` loader removed | install a supported loader (`tsx`, `@swc-node/register`, `oxc`, `jiti`…), `preferTs` |
| `.env` not loaded | dotenv removed | `import 'dotenv/config'` |
| Raw result mapping errors ("value doesn't match type") | `validate`/`strict` always on; PG `sum`/`count` return strings | cast in SQL or map with a custom type |

### Context & identity map

| Symptom | Likely cause | Fix |
|---|---|---|
| Error about using the global EntityManager for context-specific actions | code runs outside a request context (cron, queue, bootstrap, event listener, test) | `@CreateRequestContext()` on an async method, or `em.fork()`; don't enable `allowGlobalContext` |
| `@CreateRequestContext` validation error | decorated method is not `async`, or no `MikroORM`/`EntityManager`/repository on the instance | make it async; inject one, or pass `() => orm` |
| Memory grows in a long job | single fork accumulating entities | flush + `em.clear()` per chunk; `em.stream()` |
| Data from another request appears / stale entity | sharing one fork across requests or parallel transactions | one fork per request/job; `clear: true` or fresh fork for parallel tx |
| Entities "detached" after an error | rollback detaches managed entities | retry on a new fork |

### Persistence

| Symptom | Likely cause | Fix |
|---|---|---|
| `persistAndFlush is not a function` | removed in v7 | `em.persist(x).flush()` or single `flush()` |
| Change not saved | entity is detached (other fork, cache, DTO), or modified after `nativeUpdate` | `em.merge()`/`em.upsert()`; or load in this fork |
| Query in same unit of work doesn't see a just-modified scalar | v7 AUTO flush doesn't detect scalar edits before queries | `em.persist(entity)` after mutating, or flush first |
| `em.flush()` inside a subscriber/hook throws | not allowed | `uow.computeChangeSet` in `onFlush`; persist in `beforeFlush` |
| Collection change doesn't trigger `beforeUpdate` | hooks fire only for scalars/owning to-one | handle `uow.getCollectionUpdates()` in a flush subscriber |
| `OptimisticLockError` | concurrent update of a versioned entity | reload + retry the command (bounded), or surface 409 |
| Typos in `em.create()` data now fail compilation | stricter v7 typing | fix the DTO mapping (good catch, not a bug) |
| Orphaned child rows remain | relying on `cascade` for DB rules | `orphanRemoval` (ORM) and/or `deleteRule: 'cascade'` (DB) |
| Times shifted by N hours after upgrade | `forceUtcTimezone` now `true` with local-time `timestamp` data | set explicitly; migrate data; use `timestamptz` |
| `JSON.stringify` crashes on ids | native `bigint` | `BigIntType('string')` or custom serializer |

### Querying & loading

| Symptom | Likely cause | Fix |
|---|---|---|
| `getKnex is not a function` / `getKnexQuery` missing | Knex removed | `em.getKysely()` / QB / `raw()`; `@mikro-orm/knex-compat` as a bridge |
| Awaiting a QB returns a builder / type error | QB is not thenable | `.getResult()` / `.execute()` |
| QB `where` rejects alias / type errors after splitting builder across statements | v7 alias tracking | chain fluently or reassign `qb = qb.leftJoin(...)` |
| Accessing a relation throws "not initialized" / returns only PK | relation not populated | add to `populate`, type return as `Loaded<>` |
| Many small queries per request (N+1) | lazy access in loops, GraphQL resolvers | `populate`, `em.populate()`, dataloader |
| Row explosion / slow query with several collections | `joined` strategy on to-many | keep `balanced`; `populateHints` per path |
| Populate hint has no effect | hint path not in `populate` | keep `populate` and `populateHints` in sync |
| Populated collection is readonly | `populateHints` `limit` makes it partial | load full collection for mutations |
| Filter not applied | Kysely/raw SQL, QB without `applyFilters()`, or insert | add predicates; RLS for guarantees |
| Parent rows "disappear" with a soft-delete filter | filtered NOT NULL M:1 → INNER JOIN | `strict`/`autoJoinRefsForFilters` settings; review relation filters |
| Read-after-write returns old data | read went to a replica | `connectionType: 'write'` or read inside the tx |
| Cached result stale after write | result cache keyed manually | `em.clearCache(key)` after writes |

### RLS & multi-tenancy

| Symptom | Likely cause | Fix |
|---|---|---|
| Tenant sees all rows | app connects as table owner/superuser; or no `rowLevelSecurity` | non-owner `app_user` role / `rowLevelSecurity: 'force'` |
| Tenant sees no rows / error on unset variable | session context not staged (job, middleware ordering), filter-bridge fail-closed | stage via guard/fork; `nullif(current_setting(..., true), '')` in hand-written policies |
| Error when setting session context | `implicitTransactions: false` / `disableTransactions`, or changing context inside a transaction | keep implicit tx; set before `begin` |
| `em.stream()` throws under RLS | `'transaction'` strategy, staged context, no tx | wrap in `em.transactional()` |
| `req.user` undefined in tenant middleware | Nest middleware runs before guards | tenant guard after auth (queries-and-transactions §11 A) |
| `RowLevelSecurityViolationException` | write fails `WITH CHECK` (wrong tenantId on insert/update) | set `tenantId` from context; map to 403/404 |
| `migration:create` wants to drop existing policies | hand-written policies not in metadata | declare them, or `schemaGenerator.ignorePolicies: true` |
| Schema-per-tenant table missing | wildcard entities skipped by schema tooling | `includeWildcardSchema: true`; `migrator.up({ schema })` per tenant |

### Migrations

| Symptom | Likely cause | Fix |
|---|---|---|
| Unexpected FK drop/re-create in first v7 migration | FK rules no longer inferred from `cascade` | declare `deleteRule`/`updateRule` to match the existing DB, or accept intentionally |
| Pivot columns renamed for self-referencing M:N | v7 derives names from `tableName` | pin `joinColumns`/`inverseJoinColumns` |
| Snapshot diff noise across environments | `migration:up` rewrites snapshot | `snapshotOnMigrate: false` (7.2) |
| `create index concurrently` fails | inside transactional migration | `isTransactional() { return false; }` in a dedicated migration |
| `getMigrator is not a function` | v7 getters | `orm.migrator`, `orm.schema`, `orm.seeder` |

---

## 2. Debugging toolkit

```ts
defineConfig({
  debug: ['query', 'query-params'],          // log SQL with params (dev only)
  // slow query logging exists in v7 — check config docs for the option name
  loggerFactory: options => new MyLogger(options),
});

em.find(Order, where, { logging: { label: 'orders.list' } });   // tag queries per call site
```

- Count queries per request in tests (custom logger collecting `query` events) to pin N+1 regressions.
- `wrap(entity).isInitialized()`, `Reference.isInitialized()`, `collection.isInitialized()` to check load state.
- `em.getUnitOfWork().getIdentityMap()` size to spot identity-map growth (diagnostics only).
- `EXPLAIN (ANALYZE, BUFFERS)` on the logged SQL; RLS predicates appear in the plan — check they use the tenant index.
- For RLS issues: `select current_user, current_setting('app.tenant', true);` inside the same transaction.

---

## 3. Code review checklist (severity-ordered)

Check in this order. Report the most severe first.

### Critical — data loss, corruption, security
- [ ] Tenant isolation: every tenant-scoped query path covered (RLS, or filter + explicit predicates on Kysely/raw/QB); inserts set `tenantId`; app role is not the owner.
- [ ] Raw SQL built with string interpolation of user input (`em.execute(\`... ${x}\`)`, `raw(\`... ${x}\`)`) → named params / bindings.
- [ ] Broker publish or external side effects inside the business transaction → outbox / after commit.
- [ ] `schema:update` or `schema.update()` used outside local/test.
- [ ] Migrations with unreviewed drops/type changes; `NOT NULL` without backfill.
- [ ] `forceUtcTimezone`/`timestamp` vs `timestamptz` mismatch on upgraded DBs.
- [ ] FK `deleteRule: 'cascade'` across aggregate boundaries (deleting a customer wipes orders).

### High — correctness bugs
- [ ] Global `orm.em` usage / missing request context in jobs and listeners.
- [ ] Detached entities mutated and expected to persist.
- [ ] `@Transactional()` assumed to create savepoints (REQUIRED); wrong propagation for audit vs outbox.
- [ ] Concurrent commands without optimistic version on the aggregate root.
- [ ] Reads that must see uncommitted writes going through replicas or AUTO-flush blind spots.
- [ ] `nativeUpdate`/bulk writes followed by reads of stale managed entities.
- [ ] Hooks calling `flush()`/`persist()`; subscribers mutating change sets without `computeChangeSet`/`recompute`.

### Medium — performance
- [ ] N+1: relation access in loops, missing `populate`, GraphQL without dataloader.
- [ ] `joined` strategy with multiple to-many populates.
- [ ] `flush()` inside loops; unbounded `findAll()`; no `em.clear()` in batch jobs.
- [ ] Missing indexes on FK/filter/tenant columns; `$or` across joins instead of `unionWhere`.
- [ ] Hydrating aggregates for list endpoints instead of projections.

### Low — v7 hygiene and maintainability
- [ ] v6 APIs (`persistAndFlush`, `getKnex`, `getMigrator`, core decorator imports, `ArrayCollection`).
- [ ] Missing `Loaded<>` return types; `any`-typed `em.create` data.
- [ ] Entities returned from controllers instead of DTOs.
- [ ] Repositories that flush; child-entity repositories.
- [ ] Implicit defaults without comments where behavior matters (load strategy, propagation, timezone).

---

## 4. Review output format

```
## Summary
<1–2 sentences: overall risk and the single most important fix>

## Findings
### [Critical] <title>  — file.ts:L42
Problem: <what is wrong and why it matters>
Fix:
<minimal code diff or snippet with imports>

### [High] ...
### [Medium] ...
### [Low] ...

## Migration impact
<schema changes required, migrations to generate, data backfills — or "none">
```

Keep each finding actionable: one problem, one fix, with the v7-correct import paths.
