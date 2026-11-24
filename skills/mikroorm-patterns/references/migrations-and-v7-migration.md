# Migrations and v6 → v7.2 Upgrade

## Contents
1. Upgrade order of operations
2. Breaking-changes checklist (v6 → v7.0)
3. Notable additions 7.1 / 7.2 worth adopting
4. Config before/after
5. Migrator setup and safe migration workflow
6. RLS policies in migrations (7.2)
7. Schema generator usage rules
8. Seeding
9. Migration anti-patterns

---

## 1. Upgrade order of operations

1. **Freeze data semantics first.** On v6, decide `forceUtcTimezone` (v7 default `true`) and explicit FK rules. Add them to the v6 config/entities and generate a migration *before* bumping packages, so the v7 diff is empty.
2. Bump runtime: Node 22.17+, TypeScript 5.8+, `module`/`moduleResolution: nodenext` (or `bundler`).
3. Bump all `@mikro-orm/*` packages to the same 7.2.x version; add `@mikro-orm/decorators` (+ `reflect-metadata` for legacy); replace `@mikro-orm/knex` with `@mikro-orm/sql` imports; add `dataloader` if you use it.
4. Fix compile errors with the checklist below (grep patterns in SKILL.md workflow E).
5. Run `npx mikro-orm migration:create` (or `migration:check`) against a copy of production schema. **Expected: no diff.** Any diff (FK rules, nullability, pivot column names, policies) is a behavior change to understand before shipping.
6. Run the full test suite against real PostgreSQL (or pglite), not SQLite.

---

## 2. Breaking-changes checklist (v6 → v7)

### Runtime & packages
- [ ] Native ESM package; CJS still works via `require(esm)`.
- [ ] Node 22.17+, TS 5.8+, `moduleResolution` `node20`/`nodenext`/`bundler`.
- [ ] `@mikro-orm/knex` → `@mikro-orm/sql`; `@mikro-orm/better-sqlite` removed.
- [ ] `dotenv` no longer loaded — `import 'dotenv/config'` yourself.
- [ ] `dataloader` is an optional peer — install explicitly.

### Decorators & metadata
- [ ] Decorators from `@mikro-orm/decorators/legacy` (or `/es`).
- [ ] `metadataProvider: ReflectMetadataProvider` (from `@mikro-orm/decorators/legacy`) + `reflect-metadata`.
- [ ] String entity references removed (`@ManyToOne('User')` → `() => User`); `checkDuplicateEntities` removed.
- [ ] Discovery options `disableDynamicFileAccess`, `requireEntitiesArray`, `alwaysAnalyseProperties` removed.
- [ ] Extensions run before discovery; use `afterDiscovered` hook to modify metadata.

### EntityManager & UoW
- [ ] `persistAndFlush` / `removeAndFlush` removed → `em.persist(x).flush()` / `em.remove(x).flush()`.
- [ ] Auto-flush no longer detects scalar changes on managed entities → `em.persist()` after mutation if a subsequent query must see it; `trackChanges` option removed. (`em.flush()` still dirty-checks.)
- [ ] `processOnCreateHooksEarly` on by default → `onCreate` runs inside `em.create()`.
- [ ] `validate` and `strict` always on → raw results must map cleanly (PG `sum` returns string).
- [ ] Stricter `em.create()` / `em.assign()` typing — typos in DTOs now fail compilation.
- [ ] `ArrayCollection` removed → `Collection`.
- [ ] Internals are `#private` (`em._xxx`, `collection.initialized`, QB `_fields`, ...) → public API.

### Querying
- [ ] `em.getKnex()` → `em.getKysely()`; `qb.getKnexQuery()` removed; knex raw → `raw()`/`sql` or `@mikro-orm/knex-compat`.
- [ ] QueryBuilder not awaitable → `execute()` / `getResult()`.
- [ ] QB alias typing: unknown aliases in `where`/`orderBy` are compile errors; top-level `$in` must be under a property.
- [ ] `raw()` returns `RawQueryFragment` → `raw<number>(...)` when assigning to a typed property.
- [ ] Raw and string keys can't be mixed in one `orderBy` object → array form.
- [ ] `.as('alias')` always quotes → pass `'alias'` not `'"alias"'`; `qb.as(Entity, 'prop')` new signature.
- [ ] `findByCursor(Entity, { where, ... })` — `where` moved into options.
- [ ] `em.addFilter({ name, cond, entity })` — single object.
- [ ] Default `loadStrategy` = `balanced`. Code that relied on `select-in` ordering or `joined` row counts may change.

### Transactions & context
- [ ] `@Transactional()` default propagation is `REQUIRED` (was `REQUIRES_NEW`). `em.transactional()` stays `NESTED`.
- [ ] `@CreateRequestContext()` only on async methods.
- [ ] Nested transaction events: `args.transaction.savepointName`; in `beforeTransactionStart`, `args.transaction` is the parent.

### Config & CLI
- [ ] `MikroORM.initSync()` removed → `new MikroORM(config)`.
- [ ] `MikroORM.init()` and `MikroOrmModule.forRoot()` require config.
- [ ] `driverOptions: { connection: { ssl } }` → `driverOptions: { ssl }`.
- [ ] `connect` option removed (always lazy).
- [ ] Env vars: explicit options > env vars > config file (`preferEnvVars: true` restores v6).
- [ ] `MIKRO_ORM_TYPE` only read by CLI; `--config` outside CLI → `MIKRO_ORM_CLI_CONFIG`.
- [ ] `ts-node` CLI loader replaced (`oxc`, `swc`, `tsx`, `jiti`, `tsimp`, 7.2 `nub`); `tsNode`/`useTsNode` → `preferTs`; `mikro-orm-esm` → `mikro-orm`.
- [ ] `globby` → native glob: **no brace expansion for entity paths**. Use arrays or `tinyglobby` (migrations/seeders paths still support braces).
- [ ] `Options` has required keys → use `Partial<Options>` for partial configs; `MikroORMOptions` removed.

### Schema, migrations, types
- [ ] `orm.getSchemaGenerator()` / `getMigrator()` / `getSeeder()` / `getEntityGenerator()` → `orm.schema` / `orm.migrator` / `orm.seeder` / `orm.entityGenerator`.
- [ ] Renames: `createSchema` → `create`, `updateSchema` → `update`, `dropSchema` → `drop`, `clearDatabase` → `clear`, `refreshDatabase` → `refresh`, `createMigration` → `create`, etc.
- [ ] `umzug` removed (built-in engine): `UmzugMigration` → `MigrationInfo`; custom storage `logMigration`/`unlogMigration` take `{ name }`.
- [ ] `Connection.loadFile` removed → read file + `executeDump()`.
- [ ] `AbstractSchemaGenerator` from `@mikro-orm/core/schema`; `DataloaderUtils` from `@mikro-orm/core/dataloader`.
- [ ] **`forceUtcTimezone: true` by default** — existing local-time data may need migrating.
- [ ] **FK `updateRule`/`deleteRule` no longer inferred from `cascade`**; set explicitly or via `schemaGenerator.defaultDeleteRule` / `defaultUpdateRule`.
- [ ] `cascade` no longer infers nullability → `nullable: true`.
- [ ] Self-referencing M:N with custom `tableName`: pivot column names derive from table name → pin with `joinColumns` / `inverseJoinColumns`.
- [ ] Arrays in object embeddables default to JSON; embedded `prefixMode` default `relative`.
- [ ] Formula/index/check/generated callbacks are `(columns, table)`; values unquoted → `quote` helper.
- [ ] Hashing is FNV-1a 64-bit; `hashAlgorithm` removed.
- [ ] Property builder: no boolean args (`.nullable()`), `.lazy()` doesn't imply `.ref()`.
- [ ] `Hidden` brand primitives only → `HiddenProps` for objects.
- [ ] Entity generator now emits `defineEntity` by default (`entityDefinition: 'entitySchema'` for old output).

---

## 3. Additions in 7.1 / 7.2 worth adopting

| Version | Feature | Use for |
|---|---|---|
| 7.1 | `LazyRef<T>` / `.lazyRef()` | PK-only typing until populated, no wrapper |
| 7.1 | PGlite driver (`@mikro-orm/pglite`) | real-PG tests in-process (`dbName: 'memory://'`) |
| 7.1 | Typed Kysely in DI projects (`discovery:export`) | typed `getKysely()` in Nest services |
| 7.2 | Native RLS (`policies`, `rowLevelSecurity`, `session`, filter `rls: true`) | multi-tenancy enforced in the DB |
| 7.2 | `.through()` to-one relations | "current/latest X" relations without denormalization |
| 7.2 | Named params in `em.execute()` / `raw()` | safe raw SQL |
| 7.2 | `await using orm` | scripts/tests cleanup |
| 7.2 | `em.map(..., { disableIdentityMap: true })` | mapping Kysely rows |
| 7.2 | `@ManyToMany({ index: true })` | index pivot FK columns |
| 7.2 | String normalization (`trim`, `case`) | emails/codes |
| 7.2 | `migrations.snapshotOnMigrate: false` | keep snapshot owned by `migration:create` |
| 7.2 | `getNativeClient()` | direct `pg.Pool` access |

---

## 4. Config before/after

```ts
// ❌ v6
import { defineConfig } from '@mikro-orm/postgresql';
export default defineConfig({
  entities: ['dist/{modules,shared}/**/*.entity.js'],  // brace expansion
  driverOptions: { connection: { ssl: { rejectUnauthorized: false } } },
  // relies on default ReflectMetadataProvider, local-time timestamps, FK rules from cascade
});

const orm = MikroORM.initSync(config);
await orm.getMigrator().up();
await orm.getSchemaGenerator().updateSchema();
```

```ts
// ✅ v7.2
import { defineConfig, PostgreSqlDriver } from '@mikro-orm/postgresql';
import { ReflectMetadataProvider } from '@mikro-orm/decorators/legacy';
import { Migrator } from '@mikro-orm/migrations';

export default defineConfig({
  driver: PostgreSqlDriver,
  metadataProvider: ReflectMetadataProvider,
  entities: ['dist/modules/**/*.entity.js', 'dist/shared/**/*.entity.js'],
  entitiesTs: ['src/modules/**/*.entity.ts', 'src/shared/**/*.entity.ts'],
  driverOptions: { ssl: { rejectUnauthorized: true } },
  forceUtcTimezone: true,
  schemaGenerator: { defaultDeleteRule: 'no action', defaultUpdateRule: 'cascade' },
  migrations: {
    path: 'dist/migrations',
    pathTs: 'src/migrations',
    snapshot: true,
    snapshotOnMigrate: false, // 7.2: only migration:create updates the snapshot
    transactional: true,
    allOrNothing: true,
  },
  extensions: [Migrator],
});

await using orm = await MikroORM.init(config);
await orm.migrator.up();
```

---

## 5. Safe migration workflow

```bash
# 1. Generate from the diff between entities and the snapshot
npx mikro-orm migration:create --name add-order-status
# first migration of an existing DB:  --initial    | hand-written: --blank

# 2. REVIEW the generated SQL (see checklist below)

# 3. Apply on a disposable / local database
npx mikro-orm migration:up

# 4. Verify nothing is left
npx mikro-orm migration:check      # must report schema is up to date
npx mikro-orm migration:pending

# 5. Prove reversibility where it matters
npx mikro-orm migration:down
npx mikro-orm migration:up
```

Review checklist for every generated migration:
- `drop table` / `drop column` — intended? data copied first?
- Column type changes (`alter column ... type`) — lossy? needs `using`? locks a large table?
- `set not null` on existing columns — backfill (`update ... set ... where ... is null`) placed **before** it in the same migration.
- FK constraint drop/re-create — `on delete` / `on update` match your intent (v7 no longer infers).
- Index creation on large tables — consider `create index concurrently` in a separate non-transactional migration (`isTransactional() { return false; }`).
- Policy drop/create (RLS) — see §6.
- Enum value additions on PG native enums.

Migration class:

```ts
import { Migration } from '@mikro-orm/migrations';

export class Migration20261124_AddOrderStatus extends Migration {
  override async up(): Promise<void> {
    this.addSql(`alter table "orders" add column "status" varchar(20) null;`);
    this.addSql(`update "orders" set "status" = 'open' where "status" is null;`);
    this.addSql(`alter table "orders" alter column "status" set not null;`);
  }

  override async down(): Promise<void> {
    this.addSql(`alter table "orders" drop column "status";`);
  }
}

export class Migration20261124_OrdersIdx extends Migration {
  override isTransactional() { return false; }
  override async up() {
    this.addSql(`create index concurrently if not exists "orders_tenant_created_idx" on "orders" ("tenant_id", "created_at");`);
  }
}
```

- `this.execute(sql)` runs immediately in the migration's transaction (use when you need results).
- Avoid `this.getEntityManager()` with current entities — future metadata changes break old migrations. Write SQL.
- Commit the migration **and** the snapshot file together.
- Production: run compiled migrations (`dist/migrations`) via CLI or a dedicated script/job before the new app version starts; not from `onModuleInit` of every replica.

Programmatic API: `orm.migrator.create()`, `up()`, `down()`, `getPending()`, `getExecuted()`, `rollup()`.

`snapshotOnMigrate` (7.2, default `true`): when `false`, `migration:up/down` don't rewrite the snapshot — prevents noisy snapshot diffs when different environments run migrations.

---

## 6. RLS policies in migrations (7.2)

Declare on the entity:

```ts
import { Entity, PrimaryKey, Property, Index } from '@mikro-orm/decorators/legacy';

@Entity({
  rowLevelSecurity: 'force', // true | 'force' | false
  policies: [
    {
      name: 'orders_tenant_isolation',
      command: 'all',                 // select | insert | update | delete | all
      type: 'permissive',             // or 'restrictive' (AND-combined)
      roles: ['app_user'],            // default PUBLIC
      using: cols => `${cols.tenantId} = nullif(current_setting('app.tenant', true), '')::uuid`,
      check: cols => `${cols.tenantId} = nullif(current_setting('app.tenant', true), '')::uuid`,
    },
  ],
})
export class Order {
  @PrimaryKey({ type: 'uuid' }) id!: string;
  @Index() @Property({ type: 'uuid' }) tenantId!: string;
}
```

`defineEntity({ ..., rowLevelSecurity: true, policies: [...] })` works the same.

Semantics:
- `true` → `enable row level security`; with no policies it is deny-all for non-owners.
- `'force'` → also `force row level security` (owner constrained too).
- `false` → policies created but RLS left disabled — stage policies, then flip in a later migration.
- Policies are diffed by name: changes to `using`/`check`/`roles`/`command`/`type` → `drop policy` + `create policy`. Renames show as drop + create.
- STI: policies only on the root entity. TPT: each table declares its own. Abstract base policies are inherited.
- `nullif(current_setting(..., true), '')` handles both unset variables and the `''` left on pooled connections after a previous `set_config`.

Adopting a database that already has hand-written policies — pick one **before** running schema tooling:
1. Declare them in metadata (`generate-entities` can emit them), or
2. `schemaGenerator: { ignorePolicies: true }` (create-only for existing policies).
Otherwise live-introspection diffs (`schema:update`, `migration:create` with `snapshot: false`) propose dropping them and disabling RLS. `safe` mode is not an adoption strategy.

Roles (create in a privileged migration or infra, not by the app role):

```sql
create role app_user nologin;
grant usage on schema public to app_user;
grant select, insert, update, delete on all tables in schema public to app_user;
alter default privileges in schema public grant select, insert, update, delete on tables to app_user;
-- the login role used by the app must be able to `set role app_user`
grant app_user to app_login;
```

Testing: `@mikro-orm/pglite` (`dbName: 'memory://'`) with a non-superuser role (pglite connects as superuser, which bypasses RLS); `'connection'` strategy is unavailable there. Assert both that cross-tenant reads return nothing and that cross-tenant writes throw `RowLevelSecurityViolationException`.

---

## 7. Schema generator rules

- `orm.schema.create()` / `update()` / `drop()` / `refresh()` / `clear()` — use for tests and local throwaway DBs only.
- Never `schema:update` against shared/staging/prod — it has no review step and no history.
- Integration tests: `await orm.schema.refresh()` once per suite, `orm.schema.clear()` between tests (or transactional rollback per test).
- `schemaGenerator.disableForeignKeys`, `createForeignKeyConstraints`, `ignoreSchema`, `skipTables`/views — set deliberately, document why.

---

## 8. Seeding

```ts
import { Seeder } from '@mikro-orm/seeder';
import type { EntityManager } from '@mikro-orm/core';

export class DatabaseSeeder extends Seeder {
  async run(em: EntityManager) {
    em.create(Tenant, { id: TENANT_A, name: 'Acme' });
    // seeders flush automatically at the end
  }
}
```

Run with `npx mikro-orm seeder:run` or `await orm.seeder.seed(DatabaseSeeder)`; `migration:fresh --seed` for local resets. Under RLS, seed as the owner/privileged role or set the session context for each tenant.

---

## 9. Migration anti-patterns

| Anti-pattern | Consequence | Fix |
|---|---|---|
| `orm.getMigrator()` / `getSchemaGenerator()` | removed in v7 | `orm.migrator` / `orm.schema` |
| `schema:update` in prod | unreviewed destructive DDL | `migration:create` → review → `migration:up` |
| Upgrading packages and bumping `forceUtcTimezone` default implicitly | shifted timestamps | set explicitly before upgrade; migrate data |
| Expecting `Cascade.REMOVE` to create `ON DELETE CASCADE` | FK rules silently change in diff | explicit `deleteRule` |
| Editing an already-applied migration | environments diverge | write a new migration |
| Committing migration without snapshot (or vice versa) | next diff is wrong | commit both |
| Using entities / `getEntityManager()` inside migrations | old migrations break as entities evolve | raw SQL |
| `create index concurrently` inside a transactional migration | PG error | `isTransactional() → false`, separate migration |
| Brace globs for entity paths | entities silently not discovered | arrays / explicit classes |
| `driverOptions.connection` | ignored/invalid in v7 | flat `driverOptions` |
| Running migrations on every replica boot | races, partial deploys | one migration job per release |
| Hand-written RLS policies next to metadata policies | diff drops them | declare in metadata or `ignorePolicies` |
| Testing RLS migrations as superuser | policies bypassed, false green | non-owner role, `'force'` where needed |
