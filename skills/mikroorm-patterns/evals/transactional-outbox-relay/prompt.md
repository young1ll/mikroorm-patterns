---
tags: [mikroorm-patterns]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

주문 확정(placeOrder) 시 OrderPlaced 이벤트를 Kafka로 보내야 해. 지금은 서비스 메서드 안에서 flush 직후 kafka.send()를 호출하는데 가끔 DB 롤백된 주문 이벤트가 나가. MikroORM 7.2로 transactional outbox를 구현해줘. relay는 여러 인스턴스에서 돌아.
