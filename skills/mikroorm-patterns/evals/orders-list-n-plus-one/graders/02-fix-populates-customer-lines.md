---
type: llm
focus: trace
weight: 1
---

The fix populates customer, lines and lines.product (or replaces the read with a Kysely/QueryBuilder projection that joins them).
