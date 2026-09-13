# JPA, Hibernate, and JDBC Analysis

## Query-volume problems
Check for:
- N+1 reads from lazy traversal, mapper/serializer traversal, loops, streams, template rendering, or repeated repository calls.
- N+1 writes/updates.
- Repository or DB calls inside loops where bulk retrieval/update is possible.
- Repeated identical lookups inside one request/job.
- `exists`/presence checks implemented by loading full entities unnecessarily.
- unnecessary count queries.

## Fetch and object-graph problems
Check:
- inappropriate `EAGER` relationships;
- accidental lazy loading outside intended boundaries;
- multiple collection fetch joins / cartesian explosion risks;
- over-fetching full entities when projections are enough;
- wide object graphs serialized from managed entities;
- misuse of entity graphs/fetch joins.

## Write-path problems
Check:
- excessive `save()`/`saveAndFlush()` calls;
- per-row flushes;
- missing JDBC/Hibernate batching where bulk writes are expected;
- entity-by-entity deletes/updates where safe bulk operations fit;
- unnecessary merge/load-before-update patterns;
- repeated updates of the same entity in one unit of work;
- dirty-checking overhead from loading very large managed graphs.

## Result-size and pagination problems
Check:
- unbounded `findAll`/list queries on growing tables;
- large `OFFSET` pagination for deep pages;
- fetching BLOB/CLOB/large columns unnecessarily;
- in-memory filtering/sorting after broad DB reads;
- streams/cursors used without appropriate transaction/resource lifetime.

## Configuration
Inspect when relevant:
- Open Session in View;
- Hibernate batch sizes and ordered inserts/updates;
- fetch/batch-fetch sizes;
- Hikari pool sizing relative to concurrency and DB capacity;
- statement/query timeouts;
- SQL logging/statistics settings only as diagnostic aids, not permanent production fixes.

## False-positive controls
- A lazy association is not itself a defect.
- `EAGER` is not automatically wrong; judge query shape and usage.
- Bulk SQL is not automatically better if entity callbacks/versioning/invariants matter.
- A repository call in a loop may be intentional for bounded tiny sets; quantify likely impact and confidence.
