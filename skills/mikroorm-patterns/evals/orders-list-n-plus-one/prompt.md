---
tags: [mikroorm-patterns]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

주문 목록 API가 느려. 로그 보니 쿼리가 수백 개 나가. 코드 봐줘.

```ts
async list(tenantId: string) {
  const orders = await this.em.find(Order, { tenantId }, { limit: 50 });
  return orders.map(o => ({
    id: o.id,
    customer: o.customer.name,
    lines: o.lines.getItems().map(l => ({ sku: l.product.sku, qty: l.qty })),
  }));
}
```
