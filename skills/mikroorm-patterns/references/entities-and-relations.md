# Entities and Relations (MikroORM 7.2)

## Contents
1. Choosing a definition style
2. Legacy decorators (NestJS default)
3. `defineEntity` + `setClass`
4. Base entities and shared columns
5. Relations: M:1, 1:M, 1:1, M:N
6. `Ref<T>`, `LazyRef<T>`, `Rel<T>`
7. To-one via `through` (7.2)
8. Foreign key rules (`deleteRule` / `updateRule`)
9. Scalars: datetimes, bigint, enums, JSON, strings
10. Indexes, uniques, checks
11. Custom repositories
12. Common mistakes

---

## 1. Choosing a definition style

| | Legacy decorators | `defineEntity` + `setClass` |
|---|---|---|
| Import | `@mikro-orm/decorators/legacy` | `defineEntity, p` from `@mikro-orm/postgresql` (or core) |
| Metadata | `ReflectMetadataProvider` + `reflect-metadata` | Builder (no reflection) |
| Kysely types | Need `[EntityName]?: 'Name'` + explicit `entities` list | Inferred automatically |
| Domain methods | On the class | On the class extending `Schema.class` |
| Best for | Existing NestJS codebases | New modules, libraries, ESM-first, Deno/Bun |

Both can coexist in one ORM instance. Do not mix styles inside one aggregate — it makes relation typing confusing.

---

## 2. Legacy decorators

```ts
// ❌ v6
import { Entity, PrimaryKey, Property, ManyToOne } from '@mikro-orm/core';

// ✅ v7
import { Entity, PrimaryKey, Property, ManyToOne, OneToMany, Enum, Index } from '@mikro-orm/decorators/legacy';
import { Collection, type Ref, type Opt } from '@mikro-orm/core';
```

tsconfig requirements: `experimentalDecorators: true`, `emitDecoratorMetadata: true`, `module`/`moduleResolution: nodenext` (or `bundler`). The legacy `node` resolution breaks v7's `exports` maps.

```ts
import { Entity, PrimaryKey, Property, ManyToOne, OneToMany } from '@mikro-orm/decorators/legacy';
import { Collection, EntityName, type Ref, type Opt } from '@mikro-orm/core';
import { v7 as uuidv7 } from 'uuid';

@Entity({ tableName: 'orders' })
export class Order {
  [EntityName]?: 'Order'; // only needed for typed em.getKysely() with decorators

  @PrimaryKey({ type: 'uuid' })
  id: string = uuidv7();

  @ManyToOne(() => Customer, { ref: true, deleteRule: 'restrict', updateRule: 'cascade' })
  customer!: Ref<Customer>;

  @OneToMany(() => OrderLine, line => line.order, { orphanRemoval: true })
  lines = new Collection<OrderLine>(this);

  @Property({ type: 'timestamptz' })
  createdAt: Date & Opt = new Date();

  @Property({ type: 'timestamptz', onUpdate: () => new Date() })
  updatedAt: Date & Opt = new Date();
}
```

`Opt` marks properties with defaults as optional for `em.create()` typing.

---

## 3. `defineEntity` + `setClass`

```ts
import { defineEntity, p, type InferEntity } from '@mikro-orm/postgresql';

const CustomerSchema = defineEntity({
  name: 'Customer',
  properties: {
    id: p.uuid().primary(),
    email: p.string().unique(),
    name: p.string().trim(),                  // 7.2 string normalization (write + query)
    tier: p.enum(['free', 'pro']).default('free'),
    orders: () => p.oneToMany(Order).mappedBy('customer'),
    createdAt: p.datetime().onCreate(() => new Date()),
  },
});

export class Customer extends CustomerSchema.class {
  isPro() {
    return this.tier === 'pro';
  }
}
CustomerSchema.setClass(Customer); // must run at module load, before MikroORM.init()
```

```ts
const OrderSchema = defineEntity({
  name: 'Order',
  properties: {
    id: p.uuid().primary(),
    customer: () => p.manyToOne(Customer).ref().deleteRule('restrict').updateRule('cascade'),
    lines: () => p.oneToMany(OrderLine).mappedBy('order').orphanRemoval(),
    total: p.decimal().default('0'),
  },
});
export class Order extends OrderSchema.class {}
OrderSchema.setClass(Order);
```

Class-less variant (compact, but hover types get noisy):

```ts
export const Tag = defineEntity({ name: 'Tag', properties: { id: p.integer().primary(), label: p.string() } });
export type ITag = InferEntity<typeof Tag>;
```

v7 builder notes:
- Boolean args removed: `p.string().nullable()`, not `.nullable(true)`. (`persist(false)` / `autoincrement(false)` still accept `false`.)
- `.lazy()` no longer implies `.ref()` — chain `.lazy().ref()` if you want `ScalarReference`.
- Constructors must be public for `defineEntity`/`EntitySchema` classes.

---

## 4. Base entities

```ts
// decorators
import { PrimaryKey, Property } from '@mikro-orm/decorators/legacy';
import { type Opt } from '@mikro-orm/core';

export abstract class BaseEntity {
  @PrimaryKey({ type: 'uuid' })
  id: string = crypto.randomUUID();

  @Property({ type: 'timestamptz' })
  createdAt: Date & Opt = new Date();

  @Property({ type: 'timestamptz', onUpdate: () => new Date() })
  updatedAt: Date & Opt = new Date();
}
```

```ts
// defineEntity
export const BaseSchema = defineEntity({
  name: 'BaseEntity',
  abstract: true,
  properties: {
    id: p.uuid().primary(),
    createdAt: p.datetime().onCreate(() => new Date()),
    updatedAt: p.datetime().onCreate(() => new Date()).onUpdate(() => new Date()),
  },
});
const InvoiceSchema = defineEntity({ name: 'Invoice', extends: BaseSchema, properties: { /* ... */ } });
```

Do not register abstract base entities in `MikroOrmModule.forFeature()`.

---

## 5. Relations

### M:1 (owning side holds the FK)

```ts
@ManyToOne(() => Customer, { ref: true })      // NOT NULL FK, inner-joinable
customer!: Ref<Customer>;

@ManyToOne(() => User, { ref: true, nullable: true, deleteRule: 'set null' })
assignee?: Ref<User> | null;
```

### 1:M (inverse side, no column)

```ts
@OneToMany(() => OrderLine, l => l.order, { orphanRemoval: true })
lines = new Collection<OrderLine>(this);   // always initialize with `this`
```

- `orphanRemoval: true` — removing an item from the collection deletes the row at flush.
- `Collection` is the only collection class (`ArrayCollection` removed). Use `isInitialized()`, `load()`, `getItems()`, `add()`, `remove()`, `set()`, `matching()`; internals are `#private`.

### 1:1

```ts
@OneToOne(() => Profile, p => p.user, { owner: true, ref: true, deleteRule: 'cascade' })
profile!: Ref<Profile>;

// inverse side
@OneToOne(() => User, u => u.profile)
user!: Rel<User>;
```

### M:N

```ts
@ManyToMany(() => Tag, t => t.articles, { owner: true, index: true }) // index: true (7.2) indexes pivot join columns
tags = new Collection<Tag>(this);

@ManyToMany(() => Article, a => a.tags)
articles = new Collection<Article>(this);
```

When the pivot carries data (role, created_at, sort order), model it as an explicit entity with two M:1 relations and use `pivotEntity` or plain 1:M on both sides. Pivot-with-payload via hidden pivot tables becomes painful in migrations.

---

## 6. `Ref<T>`, `LazyRef<T>`, `Rel<T>`

```ts
import { type Ref, type LazyRef, type Rel, ref } from '@mikro-orm/core';
```

| Type | Runtime value | Type-level access | Use |
|---|---|---|---|
| `T` | entity (maybe uninitialized stub) | everything (lies when not loaded) | small always-loaded relations |
| `Ref<T>` (`ref: true`) | `Reference<T>` wrapper | `.id`, `.$` / `.unwrap()` when loaded, `.load()` | public aggregate boundaries, explicit loading |
| `LazyRef<T>` (7.1+) | plain entity | **PK only** until `Loaded<>` narrows it | zero-indirection with compile-time safety |
| `Rel<T>` | plain entity | everything | ESM circular imports with `emitDecoratorMetadata` |

```ts
// LazyRef
@ManyToOne(() => Author)
author!: LazyRef<Author>;
// defineEntity: author: () => p.manyToOne(Author).lazyRef()

const b1 = await em.findOneOrFail(Book, id);
b1.author.id;            // ✅
b1.author.name;          // ❌ compile error

const b2 = await em.findOneOrFail(Book, id, { populate: ['author'] });
b2.author.name;          // ✅ Loaded<Book, 'author'> narrows it
```

```ts
// Ref – assigning by id without a query
order.customer = ref(Customer, customerId);  // or em.getReference(Customer, customerId, { wrapped: true })
const c = await order.customer.load();       // loads if needed
```

---

## 7. To-one via `through` (7.2)

Read-only to-one derived from an intermediate entity, resolved by a correlated `limit 1` subquery. No column, no FK; assignments are ignored on flush.

```ts
const CaseSchema = defineEntity({
  name: 'Case',
  properties: {
    id: p.integer().primary(),
    edges: () => p.oneToMany(Edge).mappedBy('start'),
    // "the account linked through a cleared edge"
    account: () => p.manyToOne(Account).through(() => Edge).where({ state: 'cleared' }).ref().nullable(),
    // "the latest edge"
    latestEdge: () => p.oneToOne(Edge).through(() => Edge).orderBy({ id: 'desc' }).ref().nullable(),
  },
});
```

Works with both loading strategies, `Loaded<>`, serialization and `mapToPk`. Write via the through entity (`em.create(Edge, ...)`).

---

## 8. Foreign key rules

v7 no longer derives `ON DELETE` / `ON UPDATE` from `cascade`. `cascade` (`Cascade.PERSIST`, `Cascade.REMOVE`, ...) is purely Unit-of-Work behavior.

```ts
// ❌ v6 assumption: Cascade.REMOVE ⇒ ON DELETE CASCADE in the schema
@ManyToOne(() => Order, { cascade: [Cascade.REMOVE] })
order!: Order;

// ✅ v7: say what the DB should do
@ManyToOne(() => Order, { ref: true, deleteRule: 'cascade', updateRule: 'cascade' })
order!: Ref<Order>;
```

Precedence: property `deleteRule`/`updateRule` → ORM semantic defaults (FK-as-PK, M:N pivots, composite PK targets, nullable relations) → `schemaGenerator.defaultDeleteRule` / `defaultUpdateRule`.

Guidance:
- Aggregate children (order lines) → `deleteRule: 'cascade'`.
- References across aggregates (order → customer) → `'restrict'` / `'no action'`; delete through domain logic.
- Optional references → `nullable: true` + `deleteRule: 'set null'`.
- Remember `cascade` no longer infers `nullable` either.

After an upgrade, `migration:create` may emit FK drop/re-create statements — review them; they are real behavior changes.

---

## 9. Scalars

**Datetimes.** `forceUtcTimezone` is `true` by default in v7. Prefer `timestamptz` (`p.datetime()` maps to `timestamptz` on PG; with decorators set `type: 'timestamptz'` or `columnType`). Never store local time in `timestamp` without time zone.

**bigint.** Native `bigint` is the default runtime type. Choose the mode explicitly when the API layer cannot handle `bigint` (JSON.stringify throws):

```ts
@PrimaryKey()                                   // bigint ⇄ bigint
id!: bigint;

@Property({ type: new BigIntType('string') })   // bigint ⇄ string (safe for JSON)
externalId!: string;

@Property({ type: new BigIntType('number') })   // only if values < 2^53
counter!: number;
```

**Enums.** `@Enum({ items: () => Status, nativeEnumName: 'order_status' })` for PG native enums; plain string enum with check constraint otherwise. Adding values to native enums needs a migration that cannot run inside some transactions on older PG — review it.

**JSON.** `@Property({ type: 'jsonb' })` / `p.json<MyShape>()`. Query with nested objects, `$contains` / `$contained` (PG), `$elemMatch` for JSON arrays. Arrays inside object embeddables are JSON by default in v7 (`type: ArrayType` to keep old behavior).

**Strings (7.2).** `p.string().trim().uppercase()` / `new StringType({ trim: true, case: 'upper' })` normalizes on write and in query parameters (not on assignment).

**Hidden.** The `Hidden` brand only works for primitives; for `Date`/JSON use the `HiddenProps` symbol.

**Postgres aggregates.** `sum`/`count` return strings; v7 has strict validation always on, so map them with a custom type or cast in SQL.

---

## 10. Indexes, uniques, checks

```ts
@Entity()
@Index({ properties: ['tenantId', 'createdAt'] })
@Unique({ properties: ['tenantId', 'email'] })
@Check({ expression: cols => `${cols.amount} >= 0` }) // v7: callback is (columns, table), unquoted → use quote() for identifiers if needed
export class Payment {
  @Index() @Property({ type: 'uuid' }) tenantId!: string;
}
```

defineEntity: `p.string().index()`, `p.string().unique()`, plus top-level `indexes: [{ properties: [...] }]`, `uniques: [...]`.

Rules:
- Index every FK used in `where`, every column in a policy `using` clause (policies run per row), and composite `(tenant_id, …)` for tenant-scoped lists.
- Callbacks for index/check/generated columns receive `(columns, table)`; `table.qualifiedName` gives the schema-qualified name.

---

## 11. Custom repositories

```ts
import { EntityRepository } from '@mikro-orm/postgresql';
import type { Loaded } from '@mikro-orm/core';

export class OrderRepository extends EntityRepository<Order> {
  findOpenWithLines(customerId: string): Promise<Loaded<Order, 'lines'>[]> {
    return this.find(
      { customer: customerId, status: 'open' },
      { populate: ['lines'], orderBy: { createdAt: 'desc' } },
    );
  }

  // Repositories don't flush. The service owns the unit of work.
  add(order: Order) {
    this.em.persist(order);
  }
}
```

```ts
import { EntityRepositoryType } from '@mikro-orm/core';

@Entity({ repository: () => OrderRepository })
export class Order {
  [EntityRepositoryType]?: OrderRepository;
  // ...
}
```

NestJS:
- Register with `MikroOrmModule.forFeature([Order])`.
- A class named `<Entity>Repository` can be injected by class directly; otherwise `@InjectRepository(Order) repo: EntityRepository<Order>`.
- In tests mock via `getRepositoryToken(Order)` or `{ provide: OrderRepository, useValue: ... }`.
- DDD note: keep repositories as aggregate-loading/saving gateways; don't leak `QueryBuilder` into the domain layer.

---

## 12. Common mistakes

| Mistake | Fix |
|---|---|
| Decorators from `@mikro-orm/core` | `@mikro-orm/decorators/legacy` |
| No `metadataProvider` with decorators | `metadataProvider: ReflectMetadataProvider` |
| `tags: Collection<Tag>` without initializer | `tags = new Collection<Tag>(this)` |
| `@ManyToOne('Customer')` string reference | `@ManyToOne(() => Customer)` (string refs removed) |
| Relying on `cascade` for `ON DELETE` or nullability | explicit `deleteRule` / `nullable` |
| `p.string().nullable(true)` | `.nullable()` |
| `.lazy()` expecting a reference wrapper | `.lazy().ref()` |
| `ArrayCollection` | `Collection` |
| Accessing `collection.initialized` / `_em` internals | public API (`isInitialized()`, `em.getUnitOfWork()`) |
| Entity without public constructor in `defineEntity` | make it public or cast (`as any`) with explicit generic |
| bigint ids in JSON responses crash | `BigIntType('string')` or serialize explicitly |
