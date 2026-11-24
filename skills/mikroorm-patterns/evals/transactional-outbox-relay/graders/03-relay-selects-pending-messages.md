---
type: llm
focus: trace
weight: 1
---

The relay selects pending messages with LockMode.PESSIMISTIC_PARTIAL_WRITE or an explicit FOR UPDATE SKIP LOCKED so multiple relay instances don't publish the same row.
