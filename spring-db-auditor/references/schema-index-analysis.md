# Schema and Index Analysis

## Compare three layers
When available, compare independently:
1. JPA/entity intent.
2. Flyway/Liquibase/schema migration definition.
3. Actual target DB definition.

Do not assume they match.

Compare:
- SQL type and Java mapping;
- VARCHAR/CHAR length;
- DECIMAL precision/scale;
- date/time/timestamp semantics;
- nullability and defaults;
- enum representation;
- primary/foreign/unique constraints;
- charset/collation when it affects comparisons/indexing;
- indexes and index-column order.

## Index analysis
For each important query, compare predicates, joins, ordering, and ranges with actual indexes.

Check for:
- missing useful indexes;
- poor composite-column order;
- prefix overlap/redundant indexes;
- duplicate indexes;
- indexes unused by the relevant access patterns;
- over-indexing of write-heavy tables;
- oversized/wide indexed columns;
- foreign-key access paths where relevant;
- indexes that do not help because of low selectivity, function/cast use, leading-column mismatch, or range placement.

## Recommendation gate
Never recommend creating an index solely because a column appears in `WHERE`, `JOIN`, or `ORDER BY`.

A strong recommendation should explain:
- target query/access pattern;
- current indexes;
- why existing indexes are insufficient;
- proposed column order;
- selectivity/cardinality evidence when available;
- expected read benefit;
- write/storage trade-off;
- plan evidence when available.

If DB evidence is absent, downgrade confidence and label the index as a candidate to validate, not a confirmed fix.
