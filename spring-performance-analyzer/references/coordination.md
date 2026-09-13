# Agent-Neutral Coordination

## Contents

1. Portability principle
2. Roles
3. Parallel execution
4. Sequential fallback
5. Optional specialist agents
6. Handoff contract
7. Reconciliation
8. Failure and partial-access behavior
9. Completion ownership

## 1. Portability principle

The skill must work with ChatGPT, Codex, Claude, Gemini, and other coding agents that can read `SKILL.md`-style instructions. Do not require any specific sub-agent API, task tool, MCP implementation, or shell feature.

Express orchestration in terms of capabilities:

- can the agent delegate isolated work?
- can delegates share a filesystem?
- can delegates access the source repository?
- can delegates access the PERF runtime?
- can results be returned to a coordinator?

If a capability is missing, degrade to sequential analysis without reducing the quality bar.

## 2. Roles

### Coordinator

Owns:

- target/scope
- mode selection
- coverage ledger
- shared finding IDs
- contradiction resolution
- final recommendation set
- completion gate

The coordinator must not simply concatenate delegate reports.

### Static investigator

Owns the complete Static procedure in `static-analysis.md` and emits:

- coverage
- findings
- runtime verification requests
- source/config evidence

### Runtime investigator

Owns the complete Runtime procedure in `runtime-analysis.md` and emits:

- runtime coverage across replicas
- findings
- static verification requests
- metrics/log/JVM/OpenShift evidence

The Static and Runtime investigators are peers. Neither is subordinate to the other's hypotheses.

## 3. Parallel execution

When source and PERF access can be delegated independently, run Static and Runtime in parallel.

Do **not** let both write `.performance/analysis.json` concurrently.

Preferred artifacts:

- `.performance/static-result.json`
- `.performance/runtime-result.json`

Then the coordinator merges/reconciles into:

- `.performance/analysis.json`

Each investigator should read an existing canonical report first to preserve stable IDs and answer prior verification requests, but must still perform its full independent pass.

## 4. Sequential fallback

When delegation is unavailable, run:

1. complete Static or Runtime pass based on available access;
2. persist/update findings;
3. run the other complete pass if access exists;
4. reconcile contradictions;
5. execute the combined completion gate.

The order may be Static-first or Runtime-first. Never imply Static must precede Runtime.

## 5. Optional specialist agents

Use specialists only for bounded deep questions after the primary inventory is established. Examples:

- JVM/GC/allocation specialist
- concurrency/virtual-thread specialist
- Kafka/messaging specialist
- HTTP/client-pool specialist
- OpenShift/cgroup resource specialist
- code-hotspot/algorithm specialist

A specialist must receive:

- the exact question/hypothesis;
- relevant execution path;
- current evidence;
- allowed data sources;
- required output fields;
- runtime safety constraints if PERF access is involved.

Do not delegate entire responsibility for coverage to a collection of specialists. Fragmentation can miss cross-component interactions.

## 6. Handoff contract

Every agent/delegate result should contain:

```text
scope
coverage additions/changes
findings created or updated
supporting evidence
counter-evidence
verification requests
unavailable evidence
decisions that require coordinator reconciliation
```

Findings should follow `findings-protocol.md` and `report-schema.md`.

Avoid free-form handoffs such as "I found executor issues" without stable IDs and evidence.

## 7. Reconciliation

The coordinator must:

1. merge findings that represent the same root cause;
2. preserve both Static and Runtime evidence;
3. identify contradictions;
4. decide whether evidence confirms, partially confirms, revises, or rejects the hypothesis;
5. update status/confidence;
6. check whether one root cause explains multiple symptoms;
7. verify recommendations still make sense after all resource interactions are considered.

Example:

Static:

`S-011 suspects Kafka listener concurrency 20 is excessive for a 2-core pod.`

Runtime:

`CPU averages 55%; 17 listener/worker threads are mostly waiting for CatalogClient HTTP connections; HTTP pending acquisitions rise with lag.`

Reconciled:

`Reject CPU oversubscription as the primary mechanism. Confirm downstream HTTP capacity/latency as the dominant constraint. Kafka concurrency may amplify waiters but is not independently proven to be the root cause.`

## 8. Failure and partial-access behavior

### Source only

Complete Static. Create precise Runtime verification requests. Do not mark Runtime unavailable evidence as a failure of Static completion.

### PERF only

Complete Runtime. Create precise Static verification requests. Do not stop because source is unavailable.

### One delegate fails

Coordinator continues the available analysis and records the missing coverage/evidence. If possible, perform the failed role sequentially.

### No shared filesystem

Delegates return structured results in their message/output. Coordinator materializes the canonical report if it can write files.

### No shell/runtime command capability

Use available connectors/APIs/observability tools. Record commands/tooling that could not be used. Do not fabricate runtime evidence.

## 9. Completion ownership

Only the coordinator declares the combined investigation complete.

An investigator or specialist may declare its assigned scope complete, but combined completion requires:

- Static completion gate if Static access exists;
- Runtime completion gate if Runtime access exists;
- cross-component interaction pass;
- contradiction reconciliation;
- shared report consistency;
- final recommendation quality review.
