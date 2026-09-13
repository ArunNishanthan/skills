# Findings Protocol

## Contents

1. Purpose
2. Finding lifecycle
3. Evidence classes
4. Confidence
5. Severity
6. Static-to-Runtime verification
7. Runtime-to-Static verification
8. Contradictions
9. Duplicate findings
10. Recommendation quality
11. Fix verification

## 1. Purpose

Findings must remain stable across Static, Runtime, repeated runs, and multiple agents. Treat the report as an evidence ledger, not a collection of prose observations.

## 2. Finding lifecycle

Use these statuses:

- `suspected` - plausible mechanism exists but evidence is incomplete.
- `needs-runtime-verification` - Static evidence is strong enough to request a specific runtime check.
- `needs-static-verification` - Runtime evidence identifies a code/config area that needs source tracing.
- `confirmed` - evidence supports the causal mechanism strongly enough to act on the recommendation.
- `partially-confirmed` - part of the mechanism is proven but a material link remains uncertain.
- `rejected` - evidence contradicts the hypothesis.
- `fixed` - a human or external workflow reports that the recommended change was implemented.
- `verified` - post-change evidence demonstrates the expected improvement without unacceptable regressions.

Do not move `suspected` directly to `verified`.

## 3. Evidence classes

Represent evidence as one or more of:

- `source` - exact code/config/dependency behavior
- `deployment-config` - resource/JVM/container configuration
- `log` - timestamped log observation
- `metric` - measured value/time series
- `thread-dump` - repeated stack/thread-state evidence
- `jfr` - CPU/allocation/lock/I/O/GC event evidence
- `trace` - distributed/local trace evidence
- `event` - OpenShift/Kubernetes restart/probe/OOM/event evidence
- `cgroup` - CPU/memory throttling/limit evidence
- `load-profile` - request/message rate, payload, concurrency, test phase
- `external` - evidence from a downstream system supplied by the user/tools

Include source location or timestamp/window when available.

## 4. Confidence

Use:

- `high` - multiple independent evidence sources support the mechanism, or direct evidence proves it.
- `medium` - mechanism is well supported but one material link is inferred.
- `low` - plausible hypothesis with substantial missing evidence.

High confidence is not the same as high severity.

A Static-only configuration mismatch can be high confidence if the mechanics are deterministic. A runtime correlation without causation may remain medium confidence even with precise metrics.

## 5. Severity

Use severity based on performance impact, not code style:

- `critical` - can cause service failure, runaway resource exhaustion, or inability to meet core workload under expected load.
- `high` - materially constrains throughput/latency or creates major instability under realistic load.
- `medium` - meaningful inefficiency or scaling risk but not currently dominant.
- `low` - limited/local impact or optimization opportunity with small expected benefit.

Do not inflate severity merely because a pattern is considered a bad practice.

## 6. Static-to-Runtime verification

Static findings should ask a precise runtime question.

Bad:

`Check this in runtime.`

Good:

`S-017 requests Runtime verification: during representative load, collect active/max workers and queue depth for catalogExecutor, thread states for its workers, outbound CatalogClient pending/active connections, and downstream p95 latency. Confirm saturation only if queue growth coincides with all workers active and downstream wait dominates worker stacks.`

Runtime must independently investigate the environment first. After that, answer Static verification requests and update the original finding rather than cloning it.

## 7. Runtime-to-Static verification

Runtime findings should identify the exact source question.

Example:

`R-009: JFR attributes 34-41% sampled CPU to ProductMapper.mapAttributes during the load plateau. Static verification requested: trace callers and determine whether this method repeatedly copies/sorts attributes or is invoked multiple times per product.`

Static must still perform its own full inventory before limiting attention to Runtime hotspots.

## 8. Contradictions

Contradiction resolution is mandatory.

Examples:

- Static predicts executor saturation, Runtime shows executor mostly idle -> reject or revise the Static hypothesis.
- Runtime blames GC, Static and Runtime together show GC is low while threads wait on an HTTP pool -> reject the GC attribution.
- Static sees an unbounded queue but Runtime shows producer rate always below drain rate -> keep as a design risk only if realistic higher load can reach it; do not call it the current bottleneck.

Record:

- the conflicting evidence;
- which conclusion changed;
- why the newer/stronger evidence wins;
- remaining uncertainty.

Use explicit language such as `rejected by runtime evidence` rather than leaving mutually incompatible findings active.

## 9. Duplicate findings

Before creating a new finding, search existing findings for the same mechanism and execution path.

Merge when:

- Static and Runtime observed the same bottleneck from different evidence;
- multiple symptoms share one root cause;
- repeated runs provide additional evidence for the same mechanism.

Keep separate findings when remediation or causal mechanisms differ even if symptoms look similar.

Prefer one root-cause finding with several evidence records over five symptom findings.

## 10. Recommendation quality

Each confirmed or partially confirmed finding should explain:

- current mechanism;
- why it causes the measured/expected impact;
- recommended change;
- why that change addresses the mechanism;
- trade-offs;
- likely next bottleneck;
- verification plan.

Do not prescribe exact pool/thread/heap values unless the available load and service-time evidence supports them. Otherwise provide a sizing method and the evidence required to choose a value.

## 11. Fix verification

This skill recommends; it does not implement fixes.

If a later run reports a fix was applied, preserve the original finding and add post-change evidence.

Example:

Before:

- queue peak: 3420
- downstream p95: 2.8 s
- end-to-end p95: 4.1 s

After:

- queue peak: 14
- downstream p95: 620 ms
- end-to-end p95: 910 ms

Mark `verified` only when evidence supports the expected mechanism improvement and important regressions have been checked.
