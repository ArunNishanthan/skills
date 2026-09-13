# Cross-Component Interaction Analysis

## Contents

1. Purpose
2. Build a capacity chain
3. Concurrency envelope
4. Queueing and service rate
5. CPU/thread interaction
6. Connection-pool interaction
7. Retry amplification
8. Memory envelope
9. Kafka/message interaction
10. Batch interaction
11. Cache interaction
12. Virtual-thread interaction
13. Evidence standard

## 1. Purpose

Many performance failures are caused by valid components combined with incompatible capacities. Analyze the relationship between limits before recommending a change to any individual component.

## 2. Build a capacity chain

For every important flow, write the constrained resources in order.

Example:

`HTTP ingress -> server execution -> application executor -> HTTP client pool -> downstream service -> response mapping -> Redis -> egress`

Another:

`Kafka partitions -> listener concurrency -> async fan-out -> executor -> DB/HTTP pool -> acknowledgement`

For each resource capture, if known:

- configured concurrency/capacity
- observed active use
- queue capacity/depth
- service time
- timeout
- retry multiplier
- CPU/memory cost per unit of work

The useful throughput of the whole path cannot exceed its slowest sustainable stage.

## 3. Concurrency envelope

Do not treat configured thread counts independently.

Estimate the maximum in-flight work created by the composition.

Examples:

- 12 Kafka listeners, each fanning out 4 async calls -> up to 48 downstream tasks before deeper fan-out.
- 200 inbound request threads feeding a 20-connection HTTP pool -> at most about 20 concurrent HTTP/1.1 requests to that host unless protocol/client behavior changes the model; the rest wait or do other work.
- 50 executor workers feeding a semaphore of 8 -> effective concurrency at the guarded operation is 8.

Use configuration as an upper bound, not observed truth. Runtime evidence should determine actual concurrency.

## 4. Queueing and service rate

When arrival rate exceeds sustainable service rate, a queue grows until latency, memory, rejection, timeout, or backpressure intervenes.

Where steady-state measurements exist, use Little's Law as a sanity check:

`average in-flight work ~= arrival rate x average time in system`

Do not force the equation onto unstable ramp-up periods.

Investigate:

- queue capacity
- average/peak queue depth
- payload retained per queued task
- rejection/backpressure policy
- time spent queued vs executing
- whether timeouts continue consuming work after callers give up

An unbounded queue can convert a throughput limit into a delayed memory incident.

## 5. CPU/thread interaction

Thread count must be interpreted by workload type.

### CPU-bound

If most workers are runnable and CPU is near the container limit, increasing threads usually increases context switching/queueing rather than throughput.

Compare:

- pod CPU limit
- effective processor count seen by JVM
- runnable application threads
- GC/compiler CPU
- CPU throttling

### Blocking I/O

More concurrency may improve utilization while threads wait, but only until another resource such as a connection pool, downstream service, memory, or rate limit becomes the bottleneck.

### Mixed

Break the path into CPU and wait phases. Do not classify the entire request by one phase.

## 6. Connection-pool interaction

For each outbound pool compare:

`caller concurrency -> pool max -> pending queue -> downstream sustainable concurrency`

Investigate hidden queues in:

- HTTP client connection acquisition
- DB pools
- Redis pools
- SDK internal pools

A 100-thread executor plus a 10-connection pool often means 90 tasks can spend time waiting, not 100 useful concurrent downstream operations.

Do not recommend increasing the pool unless:

- the downstream supports the higher concurrency;
- caller CPU/memory can support it;
- the pool is actually saturated;
- higher concurrency is expected to improve throughput rather than amplify overload.

## 7. Retry amplification

Compute combined retry layers.

If a request can be attempted once plus `r` retries, each layer can multiply downstream attempts. Nested retry policies can create multiplicative load.

Inspect retry interactions with:

- timeouts
- circuit breakers
- message redelivery
- scheduled reprocessing
- client-library automatic retry

During downstream degradation, retries can turn a small capacity loss into a larger overload. Recommend reducing/increasing retries only after identifying failure type, idempotency, timeout budget, and downstream recovery behavior.

## 8. Memory envelope

Container memory must include more than Java heap.

Reason about:

`heap + metaspace + code cache + direct/native buffers + thread stacks + JVM/native structures + agents + mmap/page effects + safety headroom`

Also account for application retention:

`queue depth x average retained task/payload bytes`

and batch/fan-out retention:

`concurrent requests x objects retained per request`

If `-Xmx` consumes nearly the full cgroup limit, native headroom is weak even if heap GC looks healthy.

## 9. Kafka/message interaction

Compare:

- partitions
- listener concurrency
- records per poll/batch
- async fan-out factor
- executor capacity
- downstream capacity
- acknowledgement/commit semantics
- retry/redelivery volume

More consumer concurrency does not help when:

- partitions are fewer than consumers;
- downstream capacity is already saturated;
- CPU is saturated;
- work is immediately queued onto a smaller executor;
- retries dominate useful work.

Lag growth means processing throughput is below arrival rate, but the interaction analysis must explain why.

## 10. Batch interaction

Compare:

- partition/grid size
- worker threads
- chunk size
- DB/HTTP/object-store concurrency
- CPU limit
- memory retained per chunk
- temporary disk capacity
- file merge/consolidation behavior

Increasing batch parallelism can make total runtime worse when it creates connection waits, GC pressure, disk contention, or downstream throttling.

## 11. Cache interaction

Analyze miss paths as load multipliers.

Potential chain:

`cache expiry -> simultaneous misses -> N callers hit downstream -> downstream slows -> callers timeout/retry -> miss amplification`

Inspect:

- TTL alignment
- single-flight/locking
- stale-while-refresh behavior if present
- local vs remote cache layers
- serialization cost
- hot-key concurrency

Do not recommend shorter or longer TTL solely from hit ratio; include staleness requirements and miss cost.

## 12. Virtual-thread interaction

Virtual threads remove the one-platform-thread-per-blocked-operation constraint; they do not remove:

- CPU limits
- connection pool limits
- downstream quotas
- locks/semaphores
- memory retained by in-flight work
- retry amplification

A migration to virtual threads can expose downstream bottlenecks sooner by allowing much higher concurrency. Treat concurrency control as a separate design concern.

Use JDK-version-appropriate pinning guidance. Do not flag every `synchronized` block as a virtual-thread scalability issue.

## 13. Evidence standard

A cross-component finding should show the chain explicitly.

Weak:

`Kafka concurrency is too high.`

Strong:

`Kafka listener concurrency is 20. Each record is offloaded to a 50-worker executor. Runtime shows 50/50 workers active, 41 waiting for a 10-connection HTTP pool, downstream p95 is 1.8 s, and Kafka lag grows while pod CPU is only 55%. The primary constraint is outbound HTTP capacity/latency; increasing Kafka concurrency would add waiters and retained messages rather than throughput.`

Every recommendation involving capacity must state what becomes the next likely bottleneck if the recommendation is applied.
