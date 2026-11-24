---
type: llm
focus: trace
weight: 1
---

The tenant context is set at a point where req.user exists (a guard or interceptor running after JwtAuthGuard, or a middleware that verifies the JWT itself) and the response does not read req.user in a plain middleware that runs before guards.
