---
tags: [mikroorm-patterns]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

우리 NestJS 프로젝트를 MikroORM 6.4에서 7.2로 올리는 중이야. 아래 코드를 v7 방식으로 고쳐줘. 바뀐 이유도 짧게 달아줘.

```ts
import { Entity, PrimaryKey, Property, ManyToOne, Cascade } from '@mikro-orm/core';
import { EntityManager } from '@mikro-orm/postgresql';

@Entity()
export class OrderLine {
  @PrimaryKey() id!: number;
  @ManyToOne(() => Order, { cascade: [Cascade.REMOVE] }) order!: Order;
  @Property() qty!: number;
}

@Injectable()
export class OrderService {
  constructor(private readonly em: EntityManager, private readonly orm: MikroORM) {}

  async addLines(order: Order, items: { qty: number }[]) {
    for (const i of items) {
      await this.em.persistAndFlush(this.em.create(OrderLine, { order, qty: i.qty }));
    }
  }

  async report() {
    const knex = this.em.getKnex();
    return knex('order_line').sum('qty');
  }

  async migrate() {
    await this.orm.getMigrator().up();
  }
}
```
