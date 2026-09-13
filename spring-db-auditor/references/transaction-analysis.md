# Transaction Analysis

## Reconstruct ownership first
For every DB execution path, identify:
- Transaction entry point and owner.
- Whether a transaction is already active at each nested call.
- Effective propagation, isolation, read-only, timeout, and rollback semantics.
- Where the transaction actually commits/rolls back.
- Whether proxies/interceptors will actually apply.

## Spring semantics to model
Understand `REQUIRED`, `REQUIRES_NEW`, `MANDATORY`, `SUPPORTS`, `NOT_SUPPORTED`, `NEVER`, `NESTED`, isolation, timeout, `readOnly`, `rollbackFor`, `noRollbackFor`, and programmatic `TransactionTemplate`/transaction-manager usage.

Do not treat annotations as proof. Check proxy boundaries, visibility/framework behavior, self-invocation, object construction outside Spring, and indirect calls.

## Thread and async boundaries
Assume transaction context does not automatically transfer across a new thread. Inspect:
- `@Async`
- `Executor` / `ExecutorService`
- `CompletableFuture`
- virtual-thread executors
- parallel streams when DB access occurs inside them
- schedulers/background callbacks

Recompute transaction context after the boundary.

## Lifecycle analysis
For sequences like:

`READ -> UPDATE -> non-DB operation -> UPDATE -> FLUSH/COMMIT`

check:
- whether the intermediate update must be externally observable;
- whether the non-DB operation is remote I/O, blocking, CPU-heavy, or long-running;
- lock duration and connection occupancy;
- explicit/implicit flushes;
- repeated loading/updating of the same row;
- rollback consequences;
- optimistic/pessimistic locking behavior;
- whether splitting or combining transactions would change correctness.

Do not label multiple updates as bad without understanding why they exist.

## Transaction-contract findings
A shared method may deliberately require a caller-owned transaction. Determine all visible callers before declaring `@Transactional` missing.

Flag when:
- callers provide inconsistent transaction context;
- a method assumes an existing transaction but does not enforce/document the contract;
- propagation unexpectedly suspends or splits a parent transaction;
- DB work occurs outside the intended transaction because of proxy or thread-boundary behavior;
- external calls hold transactions/locks open without a justified consistency need;
- oversized transaction scopes create contention, pool pressure, rollback blast radius, or retry problems.

When recommending a contract, prefer explicit semantics such as caller-owned transaction with `MANDATORY` where appropriate, library-owned transaction, or documented orchestration boundary. Do not mechanically add `@Transactional`.
