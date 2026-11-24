---
type: llm
focus: trace
weight: 1
---

The Invoice entity declares RLS via the entity metadata (policies with using/check on tenantId and rowLevelSecurity true or 'force'), not only via hand-written SQL.
