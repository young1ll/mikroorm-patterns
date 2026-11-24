# Domain Modeling and Read Models (MikroORM 7.2)

DDD-style write models (aggregates, value objects, inheritance, polymorphism) and CQRS-style read models (views, virtual entities, projections, serialization).

## Contents
1. Aggregates with MikroORM
2. Value objects: embeddables vs custom types
3. Inheritance: mapped superclass, STI, TPT
4. Polymorphic relations
5. Read-model decision table
6. Virtual entities (`expression`)
7. View entities and materialized views
8. Projections without hydration (Kysely, `fields`, raw rows)
9. Serialization and DTOs
10. CQRS wiring in NestJS
11. Anti-patterns

---

## 1. Aggregates

Principles that map cleanly onto MikroORM's Unit of Work:

| DDD rule | MikroORM mechanism |
|---|---|
| Root controls children's lifecycle | `@OneToMany(..., { orphanRemoval: true })` (+ default `Cascade.PERSIST`) and child `deleteRule: 'cascade'` |
| Reference other aggregates by identity | `Ref<T>` / `LazyRef<T>` M:1 only; no collections pointing into another aggregate |
| One repository per aggregate root | custom `EntityRepository<Root>` only for roots; children are never queried directly by app code |
| Transaction = one aggregate (ideally) | one `flush()` per command; cross-aggregate consistency via events/outbox |
| Concurrency on the aggregate | `@Property({ version: true })` on the root; touch the root when only children change |

```ts
import { Entity, PrimaryKey, Property, ManyToOne, OneToMany, Embedded } from '@mikro-orm/decorators/legacy';
import { Collection, type Ref, ref } from '@mikro-orm/core';

@Entity({ repository: () => OrderRepository })
export class Order {
  @PrimaryKey({ type: 'uuid' }) readonly id: string = crypto.randomUUID();

  @ManyToOne(() => Customer, { ref: true, deleteRule: 'restrict' })
  readonly customer: Ref<Customer>;

  @OneToMany(() => OrderLine, l => l.order, { orphanRemoval: true })
  readonly lines = new Collection<OrderLine>(this);

  @Embedded(() => Money, { prefix: 'total_' })
  total: Money = Money.zero('KRW');

  @Property() status: 'draft' | 'placed' | 'cancelled' = 'draft';

  @Property({ version: true }) version!: number;

  @Property({ type: 'timestamptz', onUpdate: () => new Date() }) updatedAt = new Date();

  constructor(customerId: string) {
    this.customer = ref(Customer, customerId);
  }

  addLine(productId: string, qty: number, unit: Money) {
    if (this.status !== 'draft') throw new DomainError('order is not editable');
    this.lines.add(new OrderLine(this, productId, qty, unit));
    this.recalculate();
  }

  removeLine(lineId: string) {
    const line = this.lines.getItems().find(l => l.id === lineId);
    if (line) this.lines.remove(line); // orphanRemoval → DELETE at flush
    this.recalculate();
  }

  private recalculate() {
    this.total = this.lines.getItems().reduce((s, l) => s.add(l.subtotal()), Money.zero('KRW'));
    this.updatedAt = new Date(); // touch root → version bump protects the whole aggregate
  }
}
```

Construction:
- Hydration does **not** run your constructor (entities are created from the prototype), so constructor invariants only apply to genuinely new objects. Use `new Order(...)` + `em.persist(order)` for aggregates with invariants.
- Use `em.create()` for simple data-holder entities and in tests/seeders. Its stricter v7 typing catches DTO typos.
- Loading an aggregate: always populate the full child graph the root needs (`populate: ['lines']`) and return `Loaded<Order, 'lines'>` so invariant methods never run on a half-loaded collection. Calling `getItems()` on an uninitialized collection throws, which is the intended safety net.

Repository shape:

```ts
export class OrderRepository extends EntityRepository<Order> {
  async load(id: string): Promise<Loaded<Order, 'lines'>> {
    return this.findOneOrFail(id, { populate: ['lines'] });
  }
  add(order: Order) { this.em.persist(order); }
  remove(order: Order) { this.em.remove(order); }
}
```

The application service (command handler) owns the transaction and the single flush.

---

## 2. Value objects

| Shape | Use | Storage |
|---|---|---|
| Multi-field, queried by parts (Address, Money with currency, Period) | `@Embeddable` | inlined columns (`address_city`), or one JSON column with `object: true` |
| Single value with validation (Email, Sku, TenantId) | custom `Type` | one column |
| List of small VOs | embedded array | always JSON |

Embeddable:

```ts
import { Embeddable, Property } from '@mikro-orm/decorators/legacy';

@Embeddable()
export class Money {
  @Property({ type: 'decimal', precision: 18, scale: 2 }) readonly amount: string;
  @Property({ length: 3 }) readonly currency: string;

  constructor(amount: string, currency: string) { this.amount = amount; this.currency = currency; }
  static zero(c: string) { return new Money('0', c); }
  add(o: Money) { /* decimal math lib */ return new Money(/* ... */ '0', this.currency); }
}
```

```ts
// defineEntity
const AddressSchema = defineEntity({
  name: 'Address',
  embeddable: true,
  properties: { street: p.string(), city: p.string(), zip: p.string() },
});
// in an entity: shippingAddress: () => p.embedded(Address).prefix('ship_')
```

Rules:
- Register embeddables in the ORM `entities` list like entities.
- Treat them immutable: replace the whole object (`order.total = order.total.add(x)`), don't mutate fields in place. Change detection compares embedded values either way, but immutable VOs keep invariants honest.
- Prefixes: `prefix: 'ship_'`, `prefix: false`; `prefixMode` is `relative` by default in v7 (nested prefixes concatenate).
- If every field is nullable, initialize (`address = new Address()`) or the property hydrates as `null`.
- `object: true` stores JSON; querying JSON parts works on PostgreSQL.
- Embedded arrays (`@Embedded(() => Phone, { array: true })`) are JSON; `{ phones: { type: 'mobile' } }` matches elements; multiple conditions must match the same element.
- Embeddables may contain `@ManyToOne` and nested embeddables.
- **Polymorphic embeddables**: one property accepts several classes, chosen by a discriminator (`@Embeddable({ discriminatorValue: 'card' })` on each, `discriminator: 'type'` on the base) — good for payment-method/notification-channel VOs.

Custom type for single-column VOs:

```ts
import { Type, type Platform, type EntityProperty } from '@mikro-orm/core';

export class EmailType extends Type<Email, string> {
  convertToDatabaseValue(v: Email | string) { return typeof v === 'string' ? v : v.value; }
  convertToJSValue(v: string) { return Email.of(v); }
  getColumnType(prop: EntityProperty, platform: Platform) { return platform.getVarcharTypeDeclarationSQL({ length: 320 }); }
  compareAsType() { return 'string'; }
}

@Property({ type: EmailType }) email!: Email;
```

Implement `toJSON()` / `fromJSON()` on custom types that appear in cursors or API payloads (7.2 cursor pagination uses them).

---

## 3. Inheritance

| Strategy | Tables | Query root | Use when |
|---|---|---|---|
| Mapped superclass (abstract base) | none for base | not queryable | shared columns (id, timestamps, tenantId) |
| STI (`discriminatorColumn`) | 1 | no joins, filter on type | subtypes share most columns, few nullable extras |
| TPT (`inheritance: 'tpt'`) | 1 per class | base query LEFT JOINs all children | subtypes have many distinct columns / constraints |

STI:

```ts
@Entity({ discriminatorColumn: 'kind', abstract: true })
export abstract class Payment { @PrimaryKey() id!: number; @Property() amount!: string; }

@Entity({ discriminatorValue: 'card' })
export class CardPayment extends Payment { @Property() last4!: string; }

@Entity({ discriminatorValue: 'transfer' })
export class TransferPayment extends Payment { @Property() bankCode!: string; }

await em.find(Payment, {});          // returns CardPayment | TransferPayment instances
await em.find(CardPayment, {});      // adds kind = 'card'
```

- `discriminatorColumn` goes on the topmost class; abstract roots need `abstract: true`.
- To filter on the type yourself, declare the property and set `discriminator: 'kind'`.
- RLS: STI policies only on the root.

TPT:

```ts
@Entity({ inheritance: 'tpt' })
export class Party { @PrimaryKey() id!: number; @Property() displayName!: string; }

@Entity() export class Person extends Party { @Property() birthDate!: Date; }
@Entity() export class Company extends Party { @Property() registrationNo!: string; }
```

TPT caveats: querying the base joins every leaf (query concrete classes when you can); hooks don't fire for rows removed by DB cascade; soft-delete filters don't see cascaded child rows; no custom persisted discriminator; STI and TPT can't be mixed in one hierarchy.

---

## 4. Polymorphic relations (v7.0+)

One relation pointing at several unrelated entity types (comments on posts/videos, attachments, activity feeds). The row stores a discriminator + id, so **there is no database FK constraint** — integrity is the application's job.

```ts
// M:1 to a union of targets
@ManyToOne(() => [Post, Video])
target!: Post | Video;

// 7.1: union-target M:N through one pivot
@ManyToMany({ entity: () => [Image, Video], pivotTable: 'attachables', discriminator: 'attachable', owner: true })
attachments = new Collection<Image | Video>(this);
```

- TPT targets are supported in polymorphic relations; filters auto-joined on polymorphic refs are scoped to fields shared by all targets.
- The `defineEntity` DSL doesn't support union-target M:N yet (decorators/EntitySchema only).
- Prefer STI/TPT when the targets genuinely share a supertype; use polymorphic relations when they don't.
- Add an index on `(discriminator, id)` columns and a cleanup job (or triggers) for orphans, since there's no FK.
- Verify option names against the current "Polymorphic relations" docs page when introducing this — it's newer API surface.

---

## 5. Read-model decision table

| Need | Tool | Hydrates entities? | Schema-managed? |
|---|---|---|---|
| Screen needs a subset of an aggregate | `em.find(..., { fields, populate })` | yes (partial) | — |
| Ad-hoc report/aggregation, query-time | virtual entity (`expression`) | yes, read-only, no identity tracking | no |
| Reusable SQL view, other tools also read it | view entity (`view: true`) | yes, read-only | yes (`CREATE VIEW` in migrations) |
| Expensive aggregation, staleness OK | materialized view (`view: { materialized: true }`, PG) | yes, read-only | yes + refresh job |
| Hot list endpoint, max throughput | Kysely / QB `execute()` → typed DTO | no | — |
| Event-sourced/denormalized projection table | normal entity updated by projector (or Kysely upserts) | as needed | yes |

Rule of thumb: commands load aggregates; queries should rarely hydrate aggregates. Reading through domain entities for list screens wastes identity-map memory and drags lazy-loading risks into the read path.

---

## 6. Virtual entities

Evaluated at query time; no table, no PK, never change-tracked.

```ts
@Entity({
  expression: (em: EntityManager) =>
    em.createQueryBuilder(Order, 'o')
      .select(['o.customer', sql`count(*)`.as('orderCount'), sql`sum(o.total_amount)`.as('revenue')])
      .where({ status: 'placed' })
      .groupBy('o.customer'),
})
export class CustomerRevenue {
  @ManyToOne(() => Customer) customer!: Customer;     // to-one allowed, populated via select-in
  @Property() orderCount!: number;
  @Property() revenue!: string;
}

const top = await em.find(CustomerRevenue, {}, { orderBy: { revenue: 'desc' }, limit: 20, populate: ['customer'] });
```

- `expression` may also be a SQL string.
- Only scalar properties and owning to-one relations; relations always load via `select-in`.
- Great for dashboards and admin lists; for heavy, repeated queries promote to a view or materialized view.

---

## 7. View entities and materialized views

```ts
import { defineEntity, p } from '@mikro-orm/postgresql';

const ActiveSubscriptionSchema = defineEntity({
  name: 'ActiveSubscription',
  tableName: 'active_subscription_v',
  view: true,                                   // without view: true it's just a virtual entity
  expression: `select s.id, s.customer_id, s.plan, s.renews_at from subscription s where s.cancelled_at is null`,
  properties: {
    id: p.uuid().primary(),
    customer: () => p.manyToOne(Customer),
    plan: p.string(),
    renewsAt: p.datetime(),
  },
});
export class ActiveSubscription extends ActiveSubscriptionSchema.class {}
ActiveSubscriptionSchema.setClass(ActiveSubscription);
```

- Schema generator emits `CREATE VIEW` / `DROP VIEW`; migrations track and diff them.
- View entities are read-only automatically; `flush()` never writes them.
- Changing the expression → migration drops and recreates the view; review dependent views.

Materialized (PostgreSQL only):

```ts
@Entity({
  tableName: 'author_stats_mv',
  view: { materialized: true },
  expression: `select a.id, a.name, count(b.id)::int as book_count from author a left join book b on b.author_id = a.id group by a.id`,
})
export class AuthorStats { @PrimaryKey() id!: number; @Property() name!: string; @Property() bookCount!: number; }
```

```ts
import { EntityManager } from '@mikro-orm/postgresql';
await em.refreshMaterializedView(AuthorStats);                       // blocks readers
await em.refreshMaterializedView(AuthorStats, { concurrently: true }); // needs a UNIQUE index on the MV
```

- Add the unique index required by `CONCURRENTLY` in a migration (`create unique index ... on author_stats_mv (id)`).
- Refresh from a scheduled job (`@Cron` + `@CreateRequestContext`) or after the projector's batch, not per request.
- Under RLS: views run with the owner's privileges by default (policies of the base tables may be bypassed). Use `security_invoker` views (PG 15+) via a migration, or keep tenant filtering in the view entity's queries.

---

## 8. Projections without hydration

```ts
export interface OrderListItem { id: string; status: string; customerName: string; total: string; placedAt: Date }

async listOrders(tenantId: string, cursor?: string): Promise<OrderListItem[]> {
  return this.em.getKysely({ tableNamingStrategy: 'entity', columnNamingStrategy: 'property', convertValues: true })
    .selectFrom('Order as o')
    .innerJoin('Customer as c', 'c.id', 'o.customer')
    .select(['o.id', 'o.status', 'c.name as customerName', 'o.totalAmount as total', 'o.placedAt'])
    .where('o.status', '!=', 'draft')
    .orderBy('o.placedAt', 'desc')
    .limit(50)
    .execute();
}
```

Other non-hydrating options:
- `em.createQueryBuilder(Order, 'o').select([...]).execute('all')` → raw rows (`mapResults: false` for column names).
- `em.find(Order, where, { fields: ['id', 'status', 'customer.name'], disableIdentityMap: true })` → partial entities that don't pollute the identity map.
- `em.countBy(Order, 'status')` for grouped counts; `loadCount({ dataloader: true })` to batch counts.
- Need entities from Kysely rows → `em.map(Order, row, { disableIdentityMap: true })`.

Kysely respects the EM's transaction and RLS session context, so tenant isolation still applies to projections.

---

## 9. Serialization and DTOs

Default `JSON.stringify(entity)` uses ORM serialization: unwraps `Ref`/`Collection`, includes only populated relations, prunes unloaded partial fields, unpopulated relations become their PK (or `{ id }` with `forceObject`).

Explicit, shape-controlled output:

```ts
import { serialize, wrap, type EntityDTO, type Loaded } from '@mikro-orm/core';

const order = await em.findOneOrFail(Order, id, { populate: ['customer', 'lines'] });

const dto = serialize(order, {
  populate: ['customer', 'lines'],
  exclude: ['customer.passwordHash', 'version'],
  forceObject: true,     // unpopulated relations as { id }
  skipNull: true,
});

type OrderResponse = EntityDTO<Loaded<Order, 'customer' | 'lines'>>;
```

Property-level controls:
- `hidden: true` (+ `[HiddenProps]?: 'passwordHash' | 'internalNotes'` for typing objects/Dates; `Hidden<T>` for primitives).
- `serializedName: 'authorName'`, `serializer: v => v.toUpperCase()`.
- `groups: ['admin']` → only emitted when `serialize(x, { groups: ['admin'] })`.
- `serialize(x, { fields: [...] })` (7.1) is an allowlist; PKs must be listed.
- `toPOJO()` keeps hidden props and expands relations — for caching/internal use, never for API responses.

NestJS: `ClassSerializerInterceptor` can't see through `Ref`/`Collection`. Map with `serialize()` or explicit DTO mappers in the controller/presenter. Never return managed entities directly from controllers in CQRS-style code — map to response DTOs so schema changes don't leak into the API.

---

## 10. CQRS wiring in NestJS

```ts
@CommandHandler(PlaceOrder)
export class PlaceOrderHandler {
  constructor(private readonly orders: OrderRepository, private readonly em: EntityManager) {}

  @Transactional()
  async execute(cmd: PlaceOrder) {
    const order = await this.orders.load(cmd.orderId);   // aggregate, Loaded<Order,'lines'>
    order.place();                                        // domain logic, records events
    // flushed on commit; outbox rows created in the same tx (see events-tenancy-and-ops.md)
  }
}

@QueryHandler(ListOrders)
export class ListOrdersHandler {
  constructor(private readonly em: EntityManager) {}

  async execute(q: ListOrders): Promise<OrderListItem[]> {
    const read = this.em.fork();                          // no tx context → may use a read replica
    return read.getKysely(/* ... */).selectFrom('Order')/* ... */.execute();
  }
}
```

- Query side: read replicas (`replicas` config) are used automatically outside transactions; force `connectionType: 'write'` for read-after-write screens.
- Query side never calls `flush()`. Consider `em.fork({ flushMode: FlushMode.COMMIT })` for query forks to avoid accidental auto-flushes.
- Projection tables updated asynchronously (from outbox/events) are ordinary entities owned by the projector module; rebuildable via a replay job.

---

## 11. Anti-patterns

| Anti-pattern | Fix |
|---|---|
| `@OneToMany` collections pointing into another aggregate | reference by `Ref`/id; query via the other aggregate's repository |
| Repositories for child entities (`OrderLineRepository`) used by app services | go through the root |
| Mutating embeddable fields in place on shared instances | replace with a new VO |
| Domain invariants that depend on the constructor running on load | hydration skips constructors; validate in methods |
| Hydrating aggregates for list endpoints | Kysely/QB projections or views |
| Returning entities from controllers | `serialize()` / DTO mappers |
| Materialized view refreshed per request | scheduled/batch refresh, `concurrently` with unique index |
| Views under RLS assumed to be tenant-safe | `security_invoker` or explicit tenant predicates |
| Polymorphic relation without orphan cleanup | index + cleanup job; prefer STI/TPT when a supertype exists |
| Base-class TPT queries on hot paths | query concrete subclasses |
