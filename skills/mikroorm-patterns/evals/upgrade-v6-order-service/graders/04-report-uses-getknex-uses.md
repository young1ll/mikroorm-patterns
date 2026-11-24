---
type: llm
focus: trace
weight: 1
---

report() no longer uses getKnex; it uses em.getKysely(), the QueryBuilder, or em.execute with bindings.
