# Reporting

## Status
Use exactly one:
- `AUDIT COMPLETE` — every discovered ledger item reconciled.
- `AUDIT PARTIAL` — at least one discovered/relevant area could not be analyzed.

## Confidence
- `CONFIRMED` — directly supported by code semantics and/or actual DB evidence sufficient for the claim.
- `PROBABLE` — strong evidence but a material runtime/DB fact is unavailable.
- `POSSIBLE` — plausible risk that needs targeted verification.

## Severity
Use `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, or `INFO` based on likely production impact, not coding-style preference.

## Finding format
For each material finding include:

**[SEVERITY] | [CONFIDENCE] | [CATEGORY]**

- Location/path: exact code path, method, query, table, or index when known.
- Execution context: caller -> service/library -> repository/JDBC -> DB.
- Transaction context: owner, effective propagation, and relevant boundaries.
- Observation: what was found.
- Evidence: code and DB/plan evidence; distinguish inferred from observed.
- Impact: latency, query amplification, lock duration, pool pressure, correctness risk, write overhead, etc.
- Recommendation: safest high-impact fix or next verification step.
- Trade-offs: mention semantic/write/storage/consistency costs when relevant.

Avoid generic advice without linking it to evidence.

## Coverage summary
End with the ledger, for example:

```text
Entities              84/84 analyzed
Repositories          37/37 analyzed
Repository methods   211/211 analyzed
Custom queries        126/126 analyzed
DB tables              91/91 analyzed
Indexes               174/174 analyzed
Transaction roots      48/48 analyzed
DB call paths         317/317 analyzed
Shared-library calls  103/103 analyzed
```

For partial audits list every known gap and the reason, such as unavailable source, permission denied, connector limitation, generated code unavailable, or unresolved external consumer.

## Prioritization
Sort findings primarily by expected impact and confidence. Keep lower-confidence hypotheses visible in a separate validation-needed section rather than mixing them with confirmed defects.
