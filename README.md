# mikroorm-patterns

Agent Skill for **MikroORM v7.2** (NestJS + PostgreSQL): v7 package layout, `@mikro-orm/decorators/legacy` vs `defineEntity`, Unit of Work rules, `balanced` loading + `populateHints`, `Loaded<>` typing, QueryBuilder/Kysely, transactions and context isolation, native PostgreSQL row level security, v6 → v7 migration, plus DDD/CQRS, event-driven and multi-tenant patterns.

## Install

```bash
npx skills add young1ll/mikroorm-patterns
```

## Layout

```
skills/mikroorm-patterns/
├── SKILL.md                                  # hard rules, workflows, decision checklists
├── references/
│   ├── entities-and-relations.md             # decorators vs defineEntity, Ref/LazyRef/Rel, FK rules, indexes, repositories
│   ├── queries-and-transactions.md           # persist/flush, loading, QB, Kysely, stream, fork, @Transactional, RLS runtime
│   ├── migrations-and-v7-migration.md        # v6→v7 checklist, migrator workflow, RLS policies, anti-patterns
│   ├── domain-and-read-models.md             # aggregates, value objects, STI/TPT, polymorphic, views, projections, DTOs, CQRS
│   ├── events-tenancy-and-ops.md             # hooks/subscribers, audit, outbox, soft delete, tenancy strategies, bulk, cache, replicas
│   └── troubleshooting-and-review.md         # symptom→cause→fix tables, severity-ordered review checklist, report format
└── evals/                                    # 4 regression cases (v6 upgrade, RLS in Nest, outbox relay, N+1)
```

Written against MikroORM 7.2.11. Verified against the official docs: upgrading guide (v6→v7), 7.1/7.2 release posts, Row Level Security, Loading Strategies, Kysely, Transactions, Migrations, Filters, Events, Embeddables, Views, Multiple Schemas.

## Evals

Each folder under `evals/` has a prompt, an expected outcome, and one grader per assertion. With Claude Code: run `claude plugin eval . --allow-tools Write,Edit,Bash` from `skills/mikroorm-patterns/`. It compares runs with and without the skill.

## License

MIT
