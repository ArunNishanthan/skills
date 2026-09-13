# PERF Runtime Safety

## Contents

1. Scope
2. Allowed-by-default diagnostics
3. Ask before environment changes
4. Diagnostic cost classes
5. Credential and secret handling
6. Command discipline
7. Evidence retention

## 1. Scope

Runtime analysis is intended for the user's **PERF** OpenShift environment. Do not assume the same permissions or diagnostic tolerance would be acceptable in production.

Even in PERF, avoid unnecessary load and changes. The goal is diagnosis, not experimentation unless the user explicitly approves an experiment.

## 2. Allowed-by-default diagnostics

When permissions allow, these are diagnostic/read operations and may be used without asking again:

- `oc whoami`, context/project inspection
- `oc get`, `oc describe`, `oc status`
- `oc adm top pod` and container-level pod metrics
- `oc logs`, including bounded `--since` windows and `--previous` after restarts
- reading deployment/statefulset/HPA/event metadata
- `oc exec` for bounded read-only process/JVM/cgroup inspection
- `jcmd` identity/flags/heap-info/thread-dump commands
- repeated thread dumps with bounded count and interval
- short JFR captures when JFR is already available
- reading exposed Actuator/Micrometer endpoints using existing authorization
- copying a bounded diagnostic JFR file out of a pod for analysis when permitted

Diagnostic operations can still have overhead. Apply the cost guidance below.

## 3. Ask before environment changes

Explicit approval is required before actions such as:

- restarting or deleting pods
- scaling replicas up/down
- changing HPA settings
- changing ConfigMaps, Secrets, environment variables, JVM flags, manifests, deployment configuration, or routes
- redeploying/restarting to enable NMT, GC logs, JFR startup settings, or other instrumentation
- generating load or changing load-test parameters
- injecting network delay/faults
- pausing/resuming traffic
- changing Kafka consumer concurrency or application configuration
- clearing caches
- triggering batch jobs or schedulers solely for diagnosis
- taking a large heap dump
- attaching a profiler/agent that is not already present
- installing packages/tools into the container

The skill recommends fixes; it does not implement them.

## 4. Diagnostic cost classes

### Low cost

Generally safe for routine use in PERF:

- `oc get/describe/status`
- bounded log reads
- `oc adm top pod`
- `jcmd VM.version`, `VM.flags`, `VM.command_line`
- `jcmd GC.heap_info`
- single/reasonable thread dumps
- reading cgroup files
- reading existing metrics

### Moderate cost

Use only when the current hypothesis benefits from them. Keep duration/count bounded:

- several thread dumps during load
- short JFR recording with profiling settings
- broad log aggregation across many pods/containers
- class histogram
- copying diagnostic recordings

State what question the diagnostic answers before running it.

### High/disruptive cost

Ask first:

- heap dump
- long/high-detail profiling sessions
- enabling new agents/instrumentation
- repeated expensive class histograms under heavy load
- any action requiring restart/redeploy

## 5. Credential and secret handling

Prefer that the user authenticates the CLI/session outside the skill.

Never persist:

- OpenShift tokens
- passwords
- kubeconfig contents
- Secret values
- API keys
- authorization headers
- session cookies

When reading YAML or environment output, redact secret values from notes and reports.

Do not use commands that intentionally enumerate Secret contents unless the user specifically asks and the value is necessary for the performance investigation. Performance analysis should normally need secret *references*, not secret values.

## 6. Command discipline

Before each deeper runtime diagnostic:

1. state the hypothesis or evidence gap it addresses;
2. choose the lowest-cost command that can discriminate the hypothesis;
3. bound time/range/output where practical;
4. run against the confirmed PERF namespace/workload;
5. record the timestamp and target pod;
6. compare against other replicas when the observation could be pod-specific.

Avoid shotgun command execution merely because a command exists.

Examples:

- Prefer `oc logs --since=15m --timestamps` to dumping an entire multi-day log.
- Prefer a 30-60 second JFR during the bad load plateau to an open-ended recording.
- Prefer three thread dumps several seconds apart to dozens of dumps with no analysis plan.

## 7. Evidence retention

Persist only what is needed to support the finding.

- summarize logs rather than copying sensitive payloads;
- store timestamps and representative stack signatures rather than enormous raw dumps in the shared JSON;
- keep large JFR/diagnostic binaries outside the skill/report unless the user explicitly wants them retained;
- do not commit PERF diagnostic artifacts to the skills repository.
