# Query Analysis

## Reconstruct the access pattern
For each query/repository method capture, where possible:
- caller(s) and frequency context;
- filters and equality/range predicates;
- join keys;
- ordering/grouping;
- projected columns;
- expected cardinality/result limit;
- pagination mode;
- table growth/row count from DB evidence when available.

## Detect
- full or broad scans on large/hot tables;
- inefficient joins or missing join predicates;
- functions/casts on indexed columns that prevent useful access paths;
- mismatched types causing implicit conversion;
- non-sargable predicates;
- unnecessary `SELECT *`/wide entity loads;
- expensive sorting/grouping/temp-table behavior;
- deep OFFSET pagination;
- OR-heavy/dynamic query patterns whose plans degrade;
- redundant round trips that can be safely combined;
- query-per-record patterns.

## EXPLAIN reasoning
When DB-backed verification is available, use the database's read-only EXPLAIN capability. Interpret plan evidence in context: access type, chosen keys, estimated/actual rows where supported, filtering, sort/temp behavior, join order, and cardinality/selectivity.

Do not treat an EXPLAIN estimate as exact runtime truth. If runtime metrics are unavailable, state that limitation.

## Generated queries
For Spring Data derived methods, infer the likely predicate/order structure from the method and mappings; verify generated SQL or plan only when tooling permits. Do not invent exact SQL details that are not derivable.
