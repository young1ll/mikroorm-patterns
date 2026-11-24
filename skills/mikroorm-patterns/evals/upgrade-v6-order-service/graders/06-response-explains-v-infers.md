---
type: llm
focus: trace
weight: 1
---

The response explains that v7 no longer infers ON DELETE rules from cascade and adds an explicit deleteRule (e.g. 'cascade') on the ManyToOne if DB-level cascade is intended.
