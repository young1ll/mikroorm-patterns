---
tags: [mikroorm-patterns]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

NestJS + MikroORM 7.2 + PostgreSQL SaaS야. 테넌트 격리를 DB 레벨 RLS로 하고 싶어. 인증은 JwtAuthGuard가 req.user.tenantId를 채워줘. invoices 테이블(tenant_id uuid)에 정책 선언부터 요청별 컨텍스트 주입, 앱 DB 롤 구성까지 코드로 보여줘.
