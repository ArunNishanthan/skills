# Discovery and Coverage

## Goal
Build a complete inventory before deep diagnosis and maintain a ledger until every discovered item is analyzed, excluded with justification, or marked unavailable.

## Discover
- Spring Boot modules and source roots.
- Persistence configuration and profiles.
- JPA entities, embeddables, converters, inheritance mappings, relationships.
- Spring Data repositories and every repository method.
- JPQL, native SQL, named queries, Specifications/Criteria, QueryDSL, JdbcTemplate, NamedParameterJdbcTemplate, raw JDBC, stored-procedure calls.
- Flyway/Liquibase migrations and schema/bootstrap SQL.
- `@Transactional` methods/classes plus programmatic transaction APIs.
- DB-touching scheduled jobs, listeners, Kafka consumers, batch steps, commands, API paths, async tasks.
- Shared modules/libraries that contain DB access or transaction assumptions.
- Available external consumers of those shared methods.
- In DB-backed mode: tables, columns, constraints, indexes, row/cardinality statistics available from the target DB.

## Coverage ledger
Maintain counts and identities, not only totals. At minimum track:

- Entities: discovered / analyzed / unavailable
- Tables: discovered / analyzed / unavailable
- Repositories: discovered / analyzed
- Repository methods: discovered / analyzed
- Custom/native queries: discovered / analyzed
- JDBC/raw SQL access points: discovered / analyzed
- Transaction roots and annotated methods: discovered / analyzed
- DB execution paths: discovered / analyzed
- Shared-library DB methods: discovered / analyzed
- Relevant caller sites: discovered / analyzed / unavailable
- Indexes: discovered / analyzed in DB-backed mode

## Rules
- A finding does not close coverage for adjacent items.
- Sampling is not exhaustive analysis. If forced to sample because the project/tool limits access, mark the audit partial.
- Generated repository methods count as query access points even when no SQL string exists.
- Multiple call sites to the same shared DB method must be evaluated independently when transaction context differs.
- A table/entity with no apparent application path is still reconciled: classify it as unused, external-only, migration-only, or unavailable with evidence.
- Never infer 100% coverage from a successful build or from scanning only one package.
