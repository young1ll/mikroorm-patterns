---
type: llm
focus: trace
weight: 1
---

Tenant session context is applied through MikroORM (em.fork({ session }) or em.setSessionContext) with a variable such as app.tenant, not via manual set_config SQL.
