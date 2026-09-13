---
name: spring-db-auditor
description: Deeply audit Spring Boot database performance and transaction behavior across JPA/Hibernate, repositories, JDBC, shared libraries, modules, and actual database metadata. Use when analyzing Spring Boot services for N+1 queries, inefficient DB access, transaction-boundary problems, repeated updates, cross-module transaction contracts, entity/schema mismatches, indexing problems, query plans, locking, batching, pagination, or when validating suspected bottlenecks through a database MCP/tool. Require exhaustive coverage of all discovered entities, repositories, queries, transaction paths, tables, indexes, and relevant shared-library call sites before declaring the audit complete.
---

# Spring DB Auditor

Perform an evidence-driven, exhaustive audit of how a Spring Boot application interacts with its database. Reconstruct execution and transaction context before judging individual DB calls. Prefer verification against the actual database when a read-only DB MCP/tool is available.

## Core principles

1. Discover before diagnosing. Build the audit inventory first.
2. Analyze complete execution paths, not isolated repository calls.
3. Establish transaction ownership and propagation before reporting transaction defects.
4. Correlate code, generated/intended SQL, schema, indexes, and query plans.
5. Never recommend an index from a WHERE clause alone; justify it with access pattern and DB evidence when available.
6. Continue after finding severe issues. Finish every discovered audit item or explicitly mark it unavailable.
7. Separate severity from confidence. A severe hypothesis is not a confirmed defect.
8. Default to read-only analysis. Do not alter code, schema, indexes, or DB state unless the user separately asks for implementation.

## Audit workflow

1. **Select mode**
   - Static: source/config/migrations only.
   - DB-backed: static analysis plus actual DB metadata/plans through an available MCP/tool.
   - Multi-project: include all available consumers/shared libraries and trace contracts across them.
2. **Discover and ledger coverage** using [references/discovery-and-coverage.md](references/discovery-and-coverage.md).
3. **Reconstruct DB execution paths** from entry point to repository/JDBC call and through intermediate operations.
4. **Analyze transaction semantics** using [references/transaction-analysis.md](references/transaction-analysis.md).
5. **Analyze JPA/Hibernate/JDBC behavior** using [references/jpa-hibernate-analysis.md](references/jpa-hibernate-analysis.md).
6. **Analyze queries and access patterns** using [references/query-analysis.md](references/query-analysis.md).
7. **Compare entities/migrations/actual schema and indexes** using [references/schema-index-analysis.md](references/schema-index-analysis.md).
8. **Verify through DB MCP/tool when available** using [references/mcp-db-verification.md](references/mcp-db-verification.md).
9. **Analyze shared libraries and cross-project callers** using [references/cross-module-analysis.md](references/cross-module-analysis.md).
10. **Reconcile the coverage ledger.** Do not claim completion with unexplained gaps.
11. **Report findings** using [references/reporting.md](references/reporting.md).

## Mandatory reasoning model

For each suspicious DB interaction, reason in this order:

`Observation -> execution context -> transaction context -> DB evidence -> impact -> confidence -> recommendation`

Do not jump directly from suspicious-looking code to a recommendation.

For sequences such as `UPDATE -> operation -> UPDATE`, reconstruct whether the operations share a transaction, whether locks/connections remain held, whether the intermediate state is intentionally visible, whether a thread/async boundary exists, whether the second update is redundant, and what happens on failure before deciding it is problematic.

## Tool behavior

- Inspect all source roots, modules, migrations, configuration, entities, repositories, JDBC usage, and available consumers relevant to DB access.
- If a database MCP/tool exists, identify the selected environment and state it in the report. Use read-only metadata, statistics, and EXPLAIN-style operations only unless the user explicitly authorizes writes.
- If multiple DB environments/tools exist and the target is not already clear, use the environment explicitly named by the user. Otherwise analyze statically and mark DB verification pending rather than guessing.
- If source for a shared library or consumer is unavailable, report the boundary and reduce confidence. Never claim to have analyzed unavailable consumers.

## Completion gate

Do not output `AUDIT COMPLETE` until every discovered ledger category has been reconciled. Examples include entities, tables, repositories, repository methods, custom/native queries, JDBC access, transaction roots, DB call paths, shared-library DB methods, caller sites, and indexes.

If any item could not be analyzed, output `AUDIT PARTIAL` and list each gap with its reason.
