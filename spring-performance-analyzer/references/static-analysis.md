# Static Analysis Procedure

## Contents

1. Objective and stopping rule
2. Establish versions and architecture
3. Build a complete static inventory
4. Trace execution paths
5. Analyze concurrency and scheduling
6. Analyze inbound request handling
7. Analyze outbound I/O and connection pools
8. Analyze messaging and streaming
9. Analyze caching and Redis
10. Analyze serialization, transformations, and payloads
11. Analyze CPU and algorithmic cost
12. Analyze allocation and memory retention
13. Analyze synchronization and contention
14. Analyze retries, timeouts, and failure amplification
15. Analyze logging, tracing, and observability cost
16. Analyze batch and scheduled work
17. Analyze startup and bean initialization
18. Analyze JVM/container/deployment configuration
19. Analyze persistence boundaries without becoming a DB audit
20. Cross-component interaction pass
21. Static completion gate

## 1. Objective and stopping rule

The Static analyzer must explain how the code and configuration can consume CPU, memory, threads, queues, sockets, connection pools, or downstream capacity. It must not stop after pattern matching.

For each suspicious construct, continue through callers and callees until one of these conditions is reached:

- the complete performance mechanism is understood;
- the path reaches an external dependency and the remaining uncertainty is explicitly identified;
- runtime evidence is required to decide whether the risk is material.

Do not convert a possibility into a finding merely because a construct can be problematic in some applications.

## 2. Establish versions and architecture

Inspect build and runtime metadata first:

- `pom.xml`, Gradle files, dependency management, parent BOMs
- Java source/target/release level
- Spring Boot and Spring Framework versions
- Spring MVC vs WebFlux vs mixed stack
- embedded server: Tomcat, Jetty, Undertow, Netty, or custom
- messaging libraries and versions
- HTTP clients and versions
- Redis/cache clients and versions
- resilience libraries
- serialization libraries
- metrics/tracing agents and libraries
- container base image and JVM distribution
- Dockerfile, Helm, Kustomize, OpenShift templates/manifests

Version-sensitive rules must use the actual version. Examples:

- do not apply pre-JDK-24 virtual-thread monitor-pinning assumptions to JDK 24+ without checking the relevant behavior;
- do not assume Spring Boot virtual-thread pool properties behave like platform-thread executor properties;
- do not assume metric names or default pool sizes are stable across major versions.

Record unresolved versions as evidence gaps.

## 3. Build a complete static inventory

Inventory concrete components, not just packages.

### Entry points

- REST controllers and functional routes
- RPC endpoints
- Kafka/message listeners
- schedulers
- Spring Batch jobs/steps/readers/processors/writers
- command-line/application runners
- startup listeners/hooks
- file polling/import flows

### Execution resources

- `Executor`, `ExecutorService`, `TaskExecutor`, `TaskScheduler`
- `@Async`
- `CompletableFuture`
- `ForkJoinPool`
- `parallelStream()`
- explicit `Thread` creation
- virtual-thread executors and Spring virtual-thread enablement
- Reactor schedulers
- blocking queues and work queues

### External resource boundaries

- HTTP/RPC clients
- Kafka/message brokers
- Redis/cache
- persistence repositories/DAOs
- file/object storage
- DNS/TLS/network-heavy clients
- third-party SDKs

### Memory/allocation surfaces

- unbounded collections/maps/queues
- application caches
- large response/request DTOs
- byte arrays/buffers
- Jackson trees/maps and repeated object conversion
- in-memory file aggregation
- retained futures/tasks
- thread-local state

### Deployment/resource surfaces

- CPU request/limit
- memory request/limit
- replica count
- HPA configuration
- JVM flags and heap sizing
- GC selection
- native/direct memory controls
- server/client pool properties

Mark each inventory item as `analyzed`, `finding`, `needs-runtime-evidence`, `excluded-with-reason`, or `not-applicable`.

## 4. Trace execution paths

For each meaningful entry point, create an execution chain. Include nested synchronous calls and every concurrency handoff.

Example:

`Kafka listener(12) -> validator -> CompletableFuture -> pricingExecutor(40) -> WebClient -> HTTP pool(20) -> mapper -> Redis write -> ack`

At each node record:

- whether work is CPU-bound, blocking, allocation-heavy, lock-heavy, or mixed;
- concurrency introduced or constrained there;
- queue/buffer and its bound;
- timeout/retry behavior;
- payload/object amplification;
- external pool or quota consumed;
- completion/acknowledgement semantics.

Do not stop tracing because a framework abstraction hides the implementation. Follow configuration and bean wiring enough to know which executor/client/pool is actually used.

For multi-module repositories and shared libraries, continue across module boundaries when source is available. A shared library may create or select its own executor, client, queue, serializer, retry policy, or cache; inspect the actual caller context rather than assuming the library behaves identically in every service. When only a compiled dependency is available, inspect public configuration, dependency metadata, decompiled/signature information if available, and runtime evidence; otherwise record the library internals as an evidence boundary.

## 5. Analyze concurrency and scheduling

### Executors

For every executor determine:

- core/max size or virtual-thread semantics
- queue type and capacity
- rejection policy
- thread naming
- task submission sites
- whether tasks block
- nested submission to other executors
- whether callers synchronously wait using `get`, `join`, latch, semaphore, or polling
- lifecycle/shutdown behavior

Treat unbounded queues as a memory/latency risk only after finding producers that can outpace consumers.

Treat large thread counts as a problem only after classifying the workload. Many waiting threads can be acceptable for blocking I/O; many runnable CPU-bound threads on a small CPU limit usually are not.

### `CompletableFuture`

Check for:

- implicit common-pool usage
- nested futures
- `join/get` that destroys intended asynchrony
- fan-out with unbounded list size
- `allOf` retaining large result graphs
- exception paths that trigger retries or leak work
- asynchronous stages using the wrong executor

### Parallel streams

Determine whether the common pool is also used elsewhere. Look for blocking I/O inside parallel stream operations, nested parallelism, or execution from request/message threads where common-pool contention can couple unrelated workloads.

### Virtual threads

Determine JDK and Spring Boot versions first.

Analyze:

- whether blocking I/O is the dominant workload and therefore a reasonable virtual-thread fit;
- downstream connection pools/quotas that still cap concurrency;
- native/foreign calls or other version-specific pinning sources;
- explicit semaphores/bulkheads that serialize virtual threads;
- memory retained by very large numbers of concurrent tasks;
- thread-local usage at extreme concurrency;
- whether legacy executor sizing properties no longer control concurrency as expected.

Never recommend virtual threads as a generic fix for CPU saturation, downstream capacity limits, or retry storms.

### Reactive code

When WebFlux/Reactor is present, inspect for:

- blocking calls on event-loop threads
- `block()`/`toFuture().get()` in reactive paths
- inappropriate scheduler switches
- unbounded `flatMap` concurrency
- buffering/collecting entire streams
- retry operators without limits/backoff
- blocking SDKs wrapped without an appropriate bounded scheduler

## 6. Analyze inbound request handling

Trace server concurrency configuration and application work together.

Inspect, as applicable:

- server accept/backlog and connection configuration
- worker/request thread limits for platform-thread stacks
- virtual-thread enablement
- request size limits
- compression settings and CPU trade-off
- synchronous filters/interceptors
- authentication/authorization hot-path work
- request/response logging
- large body buffering
- streaming vs aggregation

Check whether inbound concurrency can exceed every downstream capacity by a large factor. High inbound concurrency is not useful if most requests immediately queue for a much smaller resource.

## 7. Analyze outbound I/O and connection pools

For each HTTP/RPC client identify the actual implementation and shared pool.

Analyze:

- max total connections and per-host/route limits
- pending-acquire/connection-request timeout
- connect timeout
- response/read timeout
- write timeout if relevant
- keep-alive and connection reuse
- idle/eviction policy
- HTTP/1.1 vs multiplexed HTTP/2 behavior where relevant
- TLS setup/reuse if connections are churned
- DNS behavior if custom
- retry behavior, including framework and client layers together
- synchronous vs async API usage

Trace all callers. Calculate whether caller-side concurrency can exceed the client pool and create hidden queueing.

Do not recommend increasing the HTTP pool until checking downstream rate limits/capacity and pod CPU/memory effects.

## 8. Analyze messaging and streaming

For Kafka and similar systems inspect:

- listener concurrency
- partitions relative to concurrency
- batch vs record listeners
- max records/batch size
- processing model: inline vs handed to another executor
- acknowledgement/commit mode
- ordering requirements
- retry/DLT/recovery behavior
- producer batching and linger/compression where relevant
- synchronous producer waits
- large-message handling
- backpressure or pause/resume logic

Build the true concurrency envelope. Example:

`listener concurrency 12 x async fan-out 4` may permit up to 48 downstream operations before considering internal queueing.

Check whether offloading work causes the listener to acknowledge or poll faster than downstream processing can safely complete.

## 9. Analyze caching and Redis

Stay focused on application performance, not Redis server internals.

Inspect:

- cache-aside/read-through/write-through pattern
- serialization format and payload size
- repeated serialization/deserialization in hot paths
- network round trips inside loops
- pipelining/batching opportunities when evidence supports them
- unbounded local caches
- TTL strategy and synchronized expiry waves
- cache stampede/single-flight behavior
- hot-key fan-in from application design
- connection pool limits
- blocking Redis calls from event-loop/reactive threads

Do not assume caching improves performance. Include invalidation, serialization cost, memory use, and miss amplification.

## 10. Analyze serialization, transformations, and payloads

Look for repeated conversions such as:

`entity -> DTO -> Map -> JSON tree -> DTO -> response`

Inspect:

- repeated `ObjectMapper.convertValue`
- `readTree`/`JsonNode` when typed streaming/binding would suffice
- large object graphs
- multiple copies of lists/maps/byte arrays
- Base64 encoding/decoding
- compression/decompression
- reflection-heavy mappers in hot loops
- sorting/filtering the same data multiple times
- materializing full collections instead of streaming/chunking

A finding should estimate amplification where possible: objects per input item, copies per payload, or bytes retained concurrently.

## 11. Analyze CPU and algorithmic cost

Inspect hot-path candidates for:

- nested loops over potentially large collections
- repeated linear scans where indexing/mapping could avoid multiplicative work
- sorting inside loops or repeated sorting
- regex compilation in frequently called code
- expensive date/time parsing or format creation
- cryptographic/compression work on request threads
- excessive string concatenation/building
- repeated normalization/parsing
- reflection/proxy-heavy loops
- accidental quadratic behavior

Do not label an O(n^2) construct critical without establishing plausible `n` and call frequency. If those are unknown, request runtime/load evidence.

## 12. Analyze allocation and memory retention

Inspect both allocation rate and retention.

### Allocation rate

- short-lived intermediate lists/maps
- boxing/unboxing in large loops
- stream pipelines that allocate heavily
- repeated DTO copying
- repeated byte/string conversions
- decompression/compression buffers
- oversized temporary arrays

### Retention

- static maps/lists
- caches without bounds
- queues that can grow faster than drain rate
- futures/tasks retaining request payloads
- thread-locals
- listeners/subscriptions never removed
- large batches retained until whole-job completion
- in-memory aggregation of files/results

Differentiate a high allocation rate causing GC pressure from a retention problem causing heap growth. Static analysis may only form a hypothesis; runtime/JFR/heap evidence should confirm material impact.

## 13. Analyze synchronization and contention

Inspect:

- broad `synchronized` methods/blocks
- `ReentrantLock` scope
- read/write locks
- semaphores
- latches/barriers
- blocking queues
- `ConcurrentHashMap.compute*` with expensive callbacks
- synchronized caches or singleton state
- global rate limiters

Trace what occurs while the lock/permit is held. I/O, logging, serialization, or downstream calls inside a critical section are much more significant than short in-memory updates.

For virtual threads, interpret monitor/pinning behavior according to the actual JDK version rather than applying a fixed rule.

## 14. Analyze retries, timeouts, and failure amplification

Find every retry layer, including:

- HTTP client retry
- resilience library retry
- message retry/re-delivery
- framework retry annotations
- manual loops
- scheduled reprocessing

Determine whether retry layers stack.

Calculate the worst-case attempt multiplier when possible. Three layers with `2` retries each can create far more downstream work than a single "2 retries" reading suggests.

Inspect:

- missing/very long timeouts
- retrying non-transient failures
- immediate retries without backoff/jitter
- retries while holding scarce resources
- retry storms during downstream degradation
- timeouts shorter than realistic downstream service time causing duplicate work

## 15. Analyze logging, tracing, and observability cost

Inspect hot paths for:

- log statements inside large loops
- eager string concatenation for disabled levels
- full request/response/body logging
- JSON serialization solely for logs
- stack traces on expected/repeated failures
- synchronous appenders
- excessive MDC construction/copying
- high-cardinality metric tags
- tracing every inner operation at extreme volume

Treat observability as workload. Estimate call frequency and payload volume before judging severity.

## 16. Analyze batch and scheduled work

Inspect:

- scheduler pool size
- overlapping executions
- fixed-rate jobs whose runtime can exceed interval
- distributed scheduling/leader-election behavior
- chunk size and in-flight chunk count
- item-level remote calls
- processor/writer parallelism
- partitioning/grid size relative to CPU and downstream capacity
- whole-file or whole-result aggregation
- temporary disk use
- retry semantics

For Spring Batch, analyze thread/concurrency configuration together with data volume and external I/O. Do not treat a larger grid size as automatically faster.

## 17. Analyze startup and bean initialization

When startup is relevant, inspect:

- expensive constructors and `@PostConstruct`
- startup database/network calls
- eager cache warmups
- broad component scanning
- classpath/dependency weight
- large configuration parsing
- sequential initialization that could block readiness
- readiness probes relative to true readiness

If startup is not part of the symptom, mark this area not-applicable rather than spending disproportionate effort.

## 18. Analyze JVM/container/deployment configuration

Read deployment and JVM settings together.

Inspect:

- CPU request vs limit
- memory request vs limit
- replica count
- heap sizing (`-Xmx`, `MaxRAMPercentage`, etc.)
- initial heap sizing where relevant
- GC selection and explicit GC flags
- metaspace limits
- direct-memory limits when configured
- thread stack size
- active processor count overrides
- container/JVM awareness
- native agents/profilers
- temporary disk limits for batch/file workloads

Reason about the full memory envelope, not heap alone:

`heap + metaspace + code cache + direct/native buffers + thread stacks + JVM/native overhead + agents + safety headroom < container limit`

A heap sized almost to the container memory limit is a likely OOM-kill risk even when Java heap itself never reaches OOM.

## 19. Analyze persistence boundaries without becoming a DB audit

The performance skill may identify application patterns such as:

- repository call per item in a large loop
- a request holding an application thread while waiting on persistence
- high configured application concurrency feeding a much smaller DB pool
- large result sets materialized in memory
- retry behavior around persistence
- connection-pool starvation symptoms inferred from configuration

Do **not** perform exhaustive index/schema/entity/query-plan auditing here. Record a handoff such as:

`Persistence amplification suspected; deep DB verification required by the database-performance analyzer.`

If a persistence problem is already proven by runtime evidence, the performance finding may still describe its application-level impact and interaction with threads/queues.

## 20. Cross-component interaction pass

After component-level analysis, perform a second pass across every major path using `interaction-analysis.md`.

At minimum compare:

- inbound concurrency vs application worker capacity
- worker concurrency vs outbound connection pools
- Kafka/listener concurrency vs async fan-out
- retries vs downstream degradation
- thread counts vs CPU limit
- heap plus native memory vs container limit
- batch parallelism vs temporary disk/network/downstream capacity
- cache miss concurrency vs downstream capacity
- queue capacity vs retained payload size

This pass is mandatory. Many real bottlenecks exist only in the relationship between otherwise valid configurations.

## 21. Static completion gate

Before finishing Static analysis:

1. Review the inventory and account for every item.
2. Review every entry point and confirm its main execution path has been traced.
3. Review every executor, queue, connection pool, retry policy, and major external client.
4. Review JVM and deployment constraints.
5. Perform the cross-component interaction pass.
6. Convert uncertain claims into explicit runtime verification requests.
7. Ensure recommendations explain mechanism, not preference.
8. List areas that could not be inspected and why.

A Static report is incomplete if it says "reviewed the main areas" without a coverage ledger.
