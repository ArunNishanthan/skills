# Shared Report Schema

## Contents

1. Canonical file
2. Concurrency rule for multiple agents
3. Top-level structure
4. Finding structure
5. Coverage structure
6. Minimal example
7. Update rules

## 1. Canonical file

When a writable workspace exists, prefer:

`.performance/analysis.json`

JSON is canonical because it is reliable for cross-agent updates. A human-readable Markdown summary may be produced separately, but it must not replace the structured evidence ledger.

If the runtime cannot write files, maintain the same conceptual structure in the agent's working context and emit it in the final result.

## 2. Concurrency rule for multiple agents

Do not let parallel agents concurrently rewrite the same JSON file.

Preferred pattern:

- Static writes `.performance/static-result.json`.
- Runtime writes `.performance/runtime-result.json`.
- Optional specialists write scoped result files such as `.performance/jvm-result.json`.
- Coordinator is the only writer of `.performance/analysis.json` during reconciliation.

Sequential single-agent runs may update the canonical file directly.

## 3. Top-level structure

Use this conceptual shape:

```json
{
  "schemaVersion": "1.0",
  "target": {
    "service": "catalog-service",
    "environment": "PERF",
    "repository": null,
    "javaVersion": "24",
    "springBootVersion": "3.x"
  },
  "investigation": {
    "startedAt": "2026-09-13T13:00:00Z",
    "updatedAt": "2026-09-13T14:00:00Z",
    "modes": {
      "static": {"status": "complete"},
      "runtime": {"status": "complete"}
    },
    "symptom": "p95 latency rises above 2s during load plateau"
  },
  "coverage": {},
  "findings": [],
  "evidenceGaps": [],
  "reconciliation": []
}
```

Use `null` when a value is unknown rather than inventing it.

## 4. Finding structure

Use fields equivalent to:

```json
{
  "id": "S-017",
  "origin": "static",
  "status": "needs-runtime-verification",
  "severity": "high",
  "confidence": "medium",
  "title": "Catalog executor can queue behind a smaller HTTP pool",
  "components": ["catalogExecutor", "CatalogClient"],
  "executionPath": [
    "CatalogKafkaListener",
    "CatalogService.process",
    "catalogExecutor",
    "CatalogClient.getProducts"
  ],
  "evidence": [
    {
      "type": "source",
      "location": "src/main/java/.../AsyncConfig.java:42",
      "observation": "executor maxPoolSize=50 and queueCapacity=5000"
    },
    {
      "type": "source",
      "location": "src/main/resources/application.yml:88",
      "observation": "CatalogClient connection pool max=10"
    }
  ],
  "mechanism": "Up to 50 workers can submit blocking calls through a pool allowing only 10 concurrent connections, so workers may accumulate in connection acquisition while the executor queue continues to accept work.",
  "impact": "Potential queueing latency and retained-task memory under sustained message load.",
  "recommendation": "Size application concurrency and outbound connection capacity as one system; do not increase either until PERF verifies downstream sustainable concurrency.",
  "justification": "Increasing executor concurrency alone cannot increase outbound throughput beyond the client/downstream constraint and can increase waiters and retained payloads.",
  "tradeoffs": [
    "A larger HTTP pool can increase downstream pressure and socket usage."
  ],
  "counterEvidence": [],
  "verificationRequests": [
    {
      "target": "runtime",
      "request": "Measure executor active/queued tasks, HTTP pool active/pending acquisition, worker thread states, and downstream p95 during representative load."
    }
  ],
  "fixVerification": "After an approved implementation change, repeat the same load and compare queue depth, HTTP pending acquisition, throughput and p95 latency."
}
```

Finding IDs should remain stable when evidence is added.

Recommended prefixes:

- `S-` Static-origin finding
- `R-` Runtime-origin finding
- `X-` Reconciled finding created only when no prior Static/Runtime finding represents the root cause

Do not rename `S-017` to a combined ID simply because Runtime confirmed it.

## 5. Coverage structure

Coverage proves exhaustiveness. Use categories containing concrete items and statuses.

Example:

```json
{
  "static": {
    "entryPoints": [
      {
        "name": "CatalogKafkaListener.onMessage",
        "status": "analyzed",
        "notes": "Traced through async executor and CatalogClient"
      }
    ],
    "executors": [
      {
        "name": "catalogExecutor",
        "status": "finding",
        "findingIds": ["S-017"]
      }
    ],
    "outboundClients": [],
    "messaging": [],
    "memory": [],
    "deployment": []
  },
  "runtime": {
    "pods": [
      {
        "name": "catalog-service-abc",
        "status": "analyzed"
      }
    ],
    "logs": [],
    "jvm": [],
    "metrics": [],
    "events": []
  }
}
```

Use these statuses:

- `analyzed`
- `finding`
- `needs-evidence`
- `excluded-with-reason`
- `not-applicable`

Avoid a generic boolean such as `reviewed: true` without concrete scope.

## 6. Minimal example

A single-mode Runtime investigation can legitimately contain:

```json
{
  "schemaVersion": "1.0",
  "target": {"service": "catalog-service", "environment": "PERF"},
  "investigation": {
    "modes": {
      "static": {"status": "not-available"},
      "runtime": {"status": "complete"}
    }
  },
  "coverage": {
    "runtime": {
      "pods": [],
      "logs": [],
      "jvm": [],
      "metrics": []
    }
  },
  "findings": [],
  "evidenceGaps": [
    "Source repository unavailable; R-003 requests Static verification of retry composition."
  ],
  "reconciliation": []
}
```

Do not treat the unavailable mode as a failure.

## 7. Update rules

1. Read the current report before updating.
2. Preserve stable IDs and existing evidence.
3. Append or amend evidence; do not erase contradictory evidence.
4. Record status changes and why they changed.
5. Merge duplicate root causes.
6. Keep rejected findings for auditability unless the user requests pruning.
7. Do not store credentials, access tokens, secret values, or full sensitive payloads.
8. Redact secrets discovered in configuration/logs before report persistence.
9. Update coverage whenever a new area is analyzed.
10. The final human summary must be derivable from the structured findings rather than introducing unsupported conclusions.
