# Runtime Analysis Procedure

## Contents

1. Objective and runtime-only completeness
2. Confirm target and permissions
3. Establish workload and symptom window
4. Inventory workloads and replicas
5. Inspect resource usage and throttling
6. Inspect logs across all replicas
7. Inspect JVM identity and configuration
8. Analyze threads and blocking
9. Analyze CPU with JFR/profiling evidence
10. Analyze heap, allocation, GC, and native memory
11. Analyze virtual threads
12. Analyze HTTP/server/client pools
13. Analyze Kafka/messaging behavior
14. Analyze Redis/cache behavior
15. Analyze metrics and traces
16. Correlate time and compare replicas
17. Form and test hypotheses
18. Runtime-only source-verification requests
19. Runtime completion gate

## 1. Objective and runtime-only completeness

Runtime analysis must independently discover performance problems from the PERF environment. Do not limit investigation to hypotheses produced by Static analysis.

A runtime observation is not automatically a root cause:

- high CPU is a symptom until hot code or runnable-thread behavior explains it;
- high memory is a symptom until heap/native/direct/cache/queue retention is separated;
- many threads are a symptom until thread states and workload type are understood;
- Kafka lag is a symptom until processing rate, downstream wait, rebalance/error behavior, or capacity explains it;
- latency is a symptom until queueing, service time, downstream time, GC, CPU, locks, or retries are separated.

Continue until the cause is defensible or the missing evidence is named.

## 2. Confirm target and permissions

Use `references/runtime-safety.md` before commands.

Verify the current OpenShift context and target namespace/project before collecting data. Prefer an existing authenticated `oc` session; never request that credentials be written into the skill or report.

Useful read-only context checks include, where available:

```sh
oc whoami
oc project
oc status
oc get deploy,statefulset,pod -o wide
```

Do not assume administrator permissions. Adapt to the access actually available.

## 3. Establish workload and symptom window

Record, if known:

- test scenario and start/end time
- target request rate or message rate
- actual request/message rate
- ramp pattern
- payload size/distribution
- test duration
- expected SLO/latency target
- known downstream dependencies
- whether the problem is throughput, latency, CPU, memory, GC, lag, errors, startup, or mixed

If the exact load profile is unavailable, continue but mark conclusions that depend on arrival rate as lower confidence.

Whenever possible, use absolute timestamps and a consistent timezone.

## 4. Inventory workloads and replicas

Inspect every replica for the target workload, not only the easiest pod to access.

Collect:

- pod name
- container names
- node
- image/tag/digest when visible
- pod age
- restart count
- readiness
- last termination reason
- CPU/memory requests and limits
- environment/JVM flags relevant to performance
- replica count and autoscaling configuration

Typical commands:

```sh
oc get pods -o wide
oc describe pod <pod>
oc get deploy <deployment> -o yaml
oc get hpa
oc get events --sort-by=.lastTimestamp
```

Avoid dumping secrets. If manifest output contains secret values, redact them from notes/report.

Check whether replicas are equivalent. A single pod with different age, node placement, restart history, or resource state can explain skew.

## 5. Inspect resource usage and throttling

When metrics are installed and permissions allow, inspect pod and container CPU/memory:

```sh
oc adm top pod -n <namespace>
oc adm top pod <pod> --containers -n <namespace>
```

One snapshot is weak evidence. Prefer multiple observations across the load window or a metrics backend with history.

### CPU

Separate:

- CPU utilization relative to pod limit
- throttling by cgroups
- runnable-thread oversubscription
- JVM GC/compiler CPU
- application hot methods
- sidecar CPU

If allowed, inspect cgroup CPU counters from inside the container. Support both cgroup v2 and v1 layouts rather than assuming one path.

For cgroup v2, useful files can include:

- `/sys/fs/cgroup/cpu.stat`
- `/sys/fs/cgroup/cpu.max`

Interpret `nr_throttled`/`throttled_usec` in context. CPU throttling is only causal if it aligns with latency/throughput degradation and the workload is actually CPU constrained.

### Memory

Separate:

- Java heap
- metaspace/code cache
- direct/native buffers
- thread stacks
- mmap/file/native library usage
- sidecars
- cgroup memory usage/events

Useful cgroup v2 files can include:

- `/sys/fs/cgroup/memory.current`
- `/sys/fs/cgroup/memory.max`
- `/sys/fs/cgroup/memory.events`

Do not infer Java heap pressure from pod RSS alone.

### Restarts/OOM

Check pod termination state and events for:

- `OOMKilled`
- probe failures
- eviction
- crash loops
- repeated restarts during load

A container OOM kill can occur while Java heap remains below `-Xmx` because native/direct/thread memory contributes to the cgroup total.

## 6. Inspect logs across all replicas

Logs are first-class evidence.

Collect the relevant time window from **all target replicas**. Include previous-container logs when a restart occurred during the symptom window.

Typical forms:

```sh
oc logs <pod> -c <container> --since=20m --timestamps
oc logs <pod> -c <container> --previous --timestamps
```

Look for patterns, not isolated messages:

- request duration/slow-operation logs
- timeout bursts
- connection-pool acquisition failures
- rejected executor tasks
- queue/backlog warnings
- Kafka rebalances, poll/commit problems, retry loops
- Redis/client timeout/reconnect behavior
- downstream 429/5xx responses
- circuit breaker transitions
- repeated stack traces
- GC logs if enabled
- OOM or allocation failures
- readiness/liveness failures
- retry storms

Calculate approximate rates/counts in the relevant window when possible. Ten thousand repeated timeouts matter differently from one startup warning.

Correlate timestamps across pods. Do not read logs from one pod and generalize to the deployment without comparison.

## 7. Inspect JVM identity and configuration

When `jcmd` is present and execution is permitted, identify the JVM process and collect low-impact diagnostics.

Possible commands:

```sh
jcmd
jcmd <pid> VM.version
jcmd <pid> VM.command_line
jcmd <pid> VM.flags
jcmd <pid> GC.heap_info
jcmd <pid> Thread.print -l
```

When virtual threads are relevant, prefer the JDK's virtual-thread-aware dump where supported:

```sh
jcmd <pid> Thread.dump_to_file -format=json /tmp/threads.json
```

Use the JDK version's `jcmd help` output to confirm command syntax. Traditional `Thread.print` and virtual-thread-aware dumps expose different information; use the one that answers the current hypothesis.

Use actual `<pid>` discovery; do not assume PID 1 even though it is common in containers.

Record:

- JDK version/vendor
- heap settings
- GC
- processor-count overrides
- virtual-thread-related configuration
- native memory tracking state if visible
- Java agents/profilers

If `jcmd` is absent, continue through metrics, logs, `/proc`, Actuator, or other available evidence. Missing tooling is not a reason to stop the entire runtime pass.

## 8. Analyze threads and blocking

Thread dumps should answer **what threads are doing**, not merely how many exist.

Prefer several bounded snapshots during the problem window rather than one dump. Compare recurring stacks.

Classify:

- RUNNABLE doing application CPU work
- RUNNABLE in native/socket operations where state semantics need care
- WAITING/TIMED_WAITING on queues, futures, latches, permits, or pools
- BLOCKED on monitors
- threads waiting for HTTP/DB/Redis connection acquisition
- executor workers idle vs saturated
- message listener threads waiting vs processing
- scheduler threads blocked by long jobs

Look for repeated stack signatures across many threads.

Examples of causal patterns:

- all executor workers blocked waiting for a 10-connection HTTP pool;
- request threads waiting on `CompletableFuture.join()` while the same constrained executor must execute the future;
- many threads blocked on one lock while the owner performs network I/O;
- CPU-saturated runnable threads repeatedly executing the same mapper/compression/regex method.

Thread count alone is not a finding.

## 9. Analyze CPU with JFR/profiling evidence

If CPU is material and JFR is available, prefer a short bounded recording in PERF.

Example form:

```sh
jcmd <pid> JFR.check
jcmd <pid> JFR.start name=perf-analysis settings=profile duration=60s filename=/tmp/performance.jfr
```

Use a shorter duration when overhead or storage is a concern. Follow runtime safety rules.

Analyze:

- method CPU samples/hot methods
- runnable-thread distribution
- allocation hotspots
- monitor/lock contention
- socket/file I/O
- GC pauses and GC CPU
- class loading/JIT if relevant
- virtual-thread events when applicable

Do not conclude that the top sampled method is automatically the root cause. Determine why it executes so often and which request/message path invokes it.

If JFR cannot be used, combine thread snapshots, process/pod CPU, metrics, logs, and source verification requests.

## 10. Analyze heap, allocation, GC, and native memory

### Heap and GC

Use available evidence such as:

- Micrometer JVM memory and GC metrics
- `jcmd <pid> GC.heap_info`
- GC logs
- JFR allocation/GC events

Distinguish:

- high allocation rate with healthy post-GC baseline;
- retention/leak-like growth where post-GC baseline rises;
- too-small heap causing frequent collections;
- long pauses due to live-set/collector behavior;
- CPU spent in GC rather than application work.

Avoid heap-dump collection by default. Heap dumps can be large and disruptive; obtain explicit approval if one is truly required.

### Native memory

If Native Memory Tracking is enabled:

```sh
jcmd <pid> VM.native_memory summary
```

If it is not enabled, do not suggest enabling it during the current run without approval because that changes JVM startup configuration and requires restart.

Correlate native memory with:

- direct buffers
- thread stacks
- class/metaspace
- code cache
- native libraries/agents
- container memory gap not explained by heap.

### Class histogram

A live class histogram can be useful but can impose cost. Use only when the memory hypothesis warrants it and PERF conditions permit. Treat it as a targeted diagnostic, not a default command.

## 11. Analyze virtual threads

First determine JDK version and whether virtual threads are actually enabled.

Do not blindly apply old pinning guidance. JDK 24 changed monitor-related virtual-thread pinning behavior; native/foreign calls and other constraints can still matter. Use version-appropriate JFR/jcmd/thread evidence.

Check:

- virtual-thread-aware thread dumps (`Thread.dump_to_file`) when supported and needed
- virtual-thread scheduler/poller diagnostics when the JDK exposes them
- virtual-thread count/rate if metrics exist
- carrier/platform-thread behavior
- long blocking operations
- native/foreign blocking
- semaphore/bulkhead/connection-pool waits
- downstream limits that virtual threads cannot remove
- memory retained per concurrent operation

A large virtual-thread population can be expected. The question is whether concurrency overwhelms a constrained downstream, memory, or CPU resource.

## 12. Analyze HTTP/server/client pools

Use metrics/config/logs/thread stacks to determine actual pool behavior.

For inbound server handling inspect, as applicable:

- active/busy request threads or event-loop saturation
- queued connections/requests if observable
- request latency distribution
- response/error rates

For outbound clients inspect:

- active/idle connections
- pending acquisition/waiters
- connection-request timeout
- connect/read/response timeout rates
- per-host limits
- connection churn
- downstream latency and error rate

A common causal pattern is:

`application concurrency > HTTP pool > pending acquire queue grows > caller latency rises > timeouts trigger retries > downstream load increases`

Prove the chain with available evidence before recommending pool changes.

## 13. Analyze Kafka/messaging behavior

Collect what the environment exposes:

- consumer lag over time
- listener/consumer concurrency
- partition assignment
- rebalance frequency
- processing time
- poll/commit errors
- retry/DLT activity
- producer send latency/errors
- downstream waits during message processing

Do not interpret lag in isolation. Determine whether arrival rate exceeds sustainable processing rate and why.

Useful causal questions:

- Are consumers CPU saturated?
- Are consumer threads mostly blocked on downstream I/O?
- Is work offloaded to an executor whose queue grows?
- Are retries consuming capacity?
- Are rebalances interrupting useful work?
- Is concurrency higher than useful partition/downstream capacity?

## 14. Analyze Redis/cache behavior

Use available client metrics/logs/traces to inspect:

- command latency
- connection-pool wait
- timeout/reconnect bursts
- request volume
- cache hit/miss behavior
- serialization cost if visible in JFR/source
- cache stampede symptoms

Do not attempt deep Redis-server tuning without server evidence and scope. Runtime analysis here is about the Spring application's use of the cache and its effect on latency/concurrency.

## 15. Analyze metrics and traces

If Actuator/Micrometer is exposed, discover available meters first rather than assuming fixed names:

```text
/actuator/metrics
```

Then inspect relevant JVM, process, system, server, executor, client, Kafka, cache, and custom metrics.

Potential evidence categories include:

- JVM heap/non-heap
- GC pause/allocation
- threads
- process/system CPU
- executor active/queued/completed
- HTTP server latency/count
- HTTP client latency/count
- connection pool active/pending
- Kafka consumer/producer metrics
- Redis/cache metrics

Metric availability and names depend on dependencies and versions. Record missing meters rather than inventing them.

If traces exist, use them to separate:

- queue/wait time
- application service time
- downstream time
- repeated/retried calls
- fan-out

Tracing is correlation evidence; sampling can hide rare paths, so combine it with other evidence.

## 16. Correlate time and compare replicas

Use one timeline for the symptom window.

Example:

`14:03 load ramp -> 14:04 HTTP pending acquisitions rise -> 14:05 executor queue rises -> 14:05:20 timeouts begin -> 14:06 retries double outbound rate -> 14:07 Kafka lag accelerates`

This is stronger than independent observations with no ordering.

Compare replicas for:

- CPU
- memory
- GC
- restarts
- error rate
- log patterns
- thread states
- traffic distribution

Investigate outliers. Possible explanations include uneven traffic, different node pressure, stale connection state, different pod age, warmup/JIT, cache skew, or degraded downstream connections.

## 17. Form and test hypotheses

For each suspected root cause write:

1. hypothesis;
2. evidence that should exist if true;
3. evidence that would contradict it;
4. cheapest safe diagnostic to discriminate;
5. result;
6. confidence update.

Example:

**Hypothesis:** executor saturation causes API latency.

Expected if true: all workers active, queue rising, workers doing useful/blocked work, request latency tracking queue depth.

Contradiction: executor mostly idle while callers wait elsewhere.

Diagnostic: executor metrics + repeated thread dumps + HTTP pool metrics.

Do not run every possible diagnostic. Let hypotheses drive deeper collection after the baseline inventory is complete.

## 18. Runtime-only source-verification requests

When source is unavailable, Runtime may still create strong findings and request Static verification.

Example:

`R-014: JFR shows 38% CPU in ProductMapper.copyAttributes reached from catalog response path. Static verification requested: inspect whether the path performs repeated collection copies or redundant transformations.`

Runtime analysis remains complete even if that source verification cannot occur now. Mark the root cause at the deepest supported evidence boundary.

## 19. Runtime completion gate

Before finishing Runtime analysis:

1. Account for all target replicas.
2. Inspect CPU/memory/resource limits and restart/events evidence.
3. Inspect logs for the relevant window across replicas.
4. Inspect JVM/thread evidence where tooling permits.
5. Investigate the dominant CPU, memory, latency, lag, or blocking mechanism rather than reporting symptoms.
6. Inspect relevant application/dependency metrics/traces if available.
7. Time-correlate the principal evidence.
8. Compare replicas and explain major outliers.
9. Record unavailable diagnostics and resulting confidence limits.
10. Produce source-verification requests for code-level unknowns when useful.

A Runtime report is incomplete if it ends at "CPU high", "memory high", "many blocked threads", or "Kafka lag high" without identifying the mechanism or evidence boundary.
