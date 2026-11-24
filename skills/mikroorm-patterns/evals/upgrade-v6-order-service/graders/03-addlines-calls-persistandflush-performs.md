---
type: llm
focus: trace
weight: 1
---

addLines no longer calls persistAndFlush and performs a single flush after the loop rather than one flush per item.
