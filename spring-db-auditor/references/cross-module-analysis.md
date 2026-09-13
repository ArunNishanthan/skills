# Cross-Module and Cross-Project Analysis

## Shared DB methods
For every shared-library method that reads/writes DB state or assumes a transaction:
- identify its intended transaction contract;
- enumerate every visible caller across available modules/projects;
- determine effective transaction context for each call site;
- identify thread/async boundaries;
- compare consistency of usage.

## Example contract pattern
A library method without `@Transactional` can be correct when all callers deliberately own the transaction. Do not flag it merely for lacking an annotation.

Flag a contract problem when one caller invokes it transactionally and another invokes it without the required context, or when the library's semantics cannot safely tolerate both.

## External/unavailable consumers
When a shared library may be consumed outside the available workspace:
- distinguish visible consumers from unknown consumers;
- do not claim the contract is globally safe;
- reduce confidence if correctness depends on unavailable callers;
- recommend an enforceable/documented contract when appropriate.

## Duplicate use across projects
The same function can have different performance/transaction implications in different projects because of caller transaction scope, data volume, concurrency, retries, and surrounding operations. Analyze each usage path, not only the shared implementation once.
