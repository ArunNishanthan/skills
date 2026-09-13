---
name: spring-performance-analyzer
description: Use when a Spring Boot or Java service has suspected or observed performance problems, when a codebase needs an exhaustive performance review before load testing, or when PERF/OpenShift runtime evidence must be correlated with source code.
---

# Spring Performance Analyzer

## Core principle

Perform a deep performance investigation, not a best-practices scan. Trace complete execution paths, reason about interacting resource limits, and continue until each relevant path reaches either a defensible root cause or an explicit evidence boundary.

Never stop because several serious findings have already been found. Depth and coverage are both required.

## Non-negotiable rules

1. **Analyze deeply.** Do not report shallow observations such as "CPU is high", "there is an executor", or "check pool sizing" as findings. Explain the mechanism that causes the performance effect.
2. **Be exhaustive.** Build an inventory and account for every relevant entry point, execution context, external dependency boundary, concurrency resource, memory-intensive path, and deployment constraint.
3. **Follow flows, not files.** Trace request, event, scheduled, and batch paths end-to-end, including synchronous and asynchronous branches.
4. **Correlate components.** Analyze interactions such as Kafka concurrency -> executor -> HTTP pool -> downstream latency -> queue growth -> pod CPU/memory.
5. **Separate evidence from inference.** Label confirmed facts, reasoned hypotheses, and missing evidence distinctly.
6. **Be version-aware.** Determine Java, Spring Boot, library, GC, and container/runtime versions before applying version-sensitive conclusions.
7. **Do not modify the application or environment.** Recommend changes only. Runtime operations that alter replicas, configuration, traffic, restarts, deployments, JVM flags, or load require explicit user approval and are outside the default analysis.
8. **Keep database auditing separate.** Identify persistence as a latency/concurrency boundary when relevant, but do not perform deep schema/index/query/entity auditing. Hand that work to a dedicated database-performance skill.
9. **Do not claim completion without coverage evidence.** Mark every inventory area as analyzed, finding, evidence-needed, excluded-with-reason, or not-applicable.
10. **Prefer root cause over symptom treatment.** Do not recommend increasing threads, concurrency, heap, replicas, or pool sizes unless the evidence shows that resource is the actual constraint and explains the downstream consequence.

## Select the analyzers

There are exactly two analyzers:

- **Static**: source repository, build files, application configuration, deployment manifests, and dependency metadata are available. Read `references/static-analysis.md` and `references/interaction-analysis.md`.
- **Runtime**: a deployed PERF environment is available. Read `references/runtime-analysis.md`, `references/runtime-safety.md`, and `references/interaction-analysis.md`.

When both are available, run both complete analyzers independently, then cross-verify and reconcile them. This is coordination of the two analyzers, not a third analysis mode. Also read `references/coordination.md` and `references/findings-protocol.md`.

Either analyzer must remain complete when the other is unavailable. Never reduce Static coverage because Runtime is available, and never limit Runtime investigation to Static hypotheses.

## Establish context before analysis

Determine, as available:

- Java/JDK version and vendor
- Spring Boot and Spring Framework versions
- MVC, WebFlux, batch, messaging, scheduling, async, and persistence technologies in use
- container image/runtime and OpenShift/Kubernetes resource limits
- workload type: HTTP, Kafka/message-driven, scheduled, batch, mixed
- expected or observed traffic/load profile
- known symptom and time window, if any
- available observability: logs, Micrometer/Actuator, Prometheus, tracing, JFR, `jcmd`, GC logs

Do not require all items before starting. Record unavailable evidence and continue with what exists.

## Build the investigation inventory

Create a coverage inventory before drawing conclusions. At minimum account for:

- inbound HTTP/RPC entry points
- Kafka/message listeners and producers
- schedulers and batch jobs
- async boundaries, executors, virtual threads, common pools, and explicit thread creation
- outbound HTTP/RPC clients and connection pools
- Redis/cache interactions
- persistence calls as external latency/concurrency boundaries
- serialization, deserialization, compression, and payload transformations
- file/network I/O
- locks, synchronization, shared mutable state, and queues
- retries, timeouts, circuit breakers, rate limits, and bulkheads
- logging and tracing hot paths
- memory-retaining structures, caches, buffers, and large collections
- startup/initialization work when startup performance matters
- JVM/GC configuration
- pod CPU/memory requests and limits, replicas, and JVM/container sizing

Use the detailed mode-specific reference to expand this inventory.

## Analyze execution paths

For each important path, construct a chain such as:

`entry point -> application stages -> async handoffs -> I/O boundaries -> queues/pools -> downstream -> completion`

For every stage determine:

- work type: CPU, blocking I/O, allocation-heavy, lock/contention, mixed
- concurrency source and maximum effective concurrency
- queue/buffer behavior and whether it is bounded
- timeout and retry behavior
- object/payload amplification
- resource dependency: CPU, heap, native/direct memory, socket, connection pool, downstream quota
- failure behavior and whether failures amplify load

If one stage delegates to another executor or pool, continue tracing. Do not stop at the first asynchronous boundary.

## Calculate interaction constraints

Use `references/interaction-analysis.md` whenever more than one concurrency, queue, memory, or downstream limit participates in a path. Quantify where evidence permits. A recommendation that changes concurrency or capacity without this reasoning is incomplete.

## Runtime safety

Runtime access is for the **PERF environment only**. Read `references/runtime-safety.md` before issuing runtime commands.

Diagnostic/read operations may be performed without additional approval when permitted by the environment, including bounded `oc` inspection, pod logs, thread dumps, `jcmd`, and short diagnostic JFR captures. Ask before any environment-changing action.

Never embed passwords, tokens, or OpenShift credentials in this skill or its report. Prefer an already authenticated CLI/session.

## Shared findings and report

Look for an existing `.performance/analysis.json` in the workspace. If present, read it as prior evidence, not as a boundary on the investigation.

If the workspace is writable, use `.performance/analysis.json` as the canonical cross-run/cross-agent report. Read `references/report-schema.md` before creating or updating it.

Every material finding must include:

- stable ID
- origin: static, runtime, or reconciled
- status
- severity and confidence
- affected execution path/components
- evidence
- causal mechanism/root cause
- observed or expected impact
- recommendation
- technical justification
- counter-evidence or uncertainty
- requested cross-verification, when useful
- verification method for a future fix

Use `references/findings-protocol.md` for status transitions and contradiction handling.

## Multi-agent and cross-runtime operation

Do not depend on a specific vendor's agent API. Read `references/coordination.md` when delegation or multiple agents are available.

Preferred shape when delegation is supported:

1. Coordinator owns scope, coverage, and final reconciliation.
2. Static investigator performs the complete Static pass.
3. Runtime investigator performs the complete Runtime pass.
4. Optional specialists may investigate bounded questions such as JVM/GC, concurrency, messaging, memory/allocation, or OpenShift resource behavior.
5. Coordinator reconciles all evidence into one final conclusion set.

If delegation is unavailable, perform the same responsibilities sequentially. The quality bar does not change.

## Recommendation standard

Do not change code or configuration. Recommend only after establishing the performance mechanism.

A strong recommendation answers:

1. What exactly is wrong?
2. What evidence supports it?
3. Through what execution path does it occur?
4. Why does it reduce throughput, increase latency, consume CPU/memory, or amplify failure?
5. Why will the proposed change address that mechanism?
6. What trade-off or new limit could the change introduce?
7. How should the improvement be verified after implementation?

Avoid generic advice such as "increase threads", "add caching", "use virtual threads", "increase heap", "add replicas", or "optimize DB" without mechanism-specific evidence.

## Completion gate

Do not declare the investigation complete until:

- the applicable inventory is fully accounted for;
- all major execution paths have been traced through their resource boundaries;
- serious findings have been drilled to root cause or an explicit evidence boundary;
- Static and Runtime contradictions have been reconciled when both modes are available;
- unsupported hypotheses are clearly marked rather than presented as facts;
- recommendations contain evidence, mechanism, justification, trade-offs, and verification guidance;
- unexamined areas and unavailable evidence are explicitly listed.

The final human-readable response should prioritize confirmed root causes first, then partially confirmed findings, then evidence-needed risks. Do not bury high-impact findings inside a generic checklist.

## References

- `references/static-analysis.md` - exhaustive source/configuration investigation procedure
- `references/runtime-analysis.md` - PERF/OpenShift/JVM runtime investigation procedure
- `references/interaction-analysis.md` - cross-component capacity and contention reasoning
- `references/findings-protocol.md` - finding lifecycle, evidence, confidence, and contradiction rules
- `references/report-schema.md` - canonical `.performance/analysis.json` structure
- `references/runtime-safety.md` - allowed diagnostics and approval boundaries
- `references/coordination.md` - agent-neutral parallel/sequential coordination model
