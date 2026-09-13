# MCP / Database Verification

## Safety
Use the DB connector/MCP in read-only mode by default. Do not execute DDL, DML, index creation, statistics mutation, or production-changing operations unless the user explicitly asks for implementation and the tool permits it.

State the target environment in the report when known.

## Useful evidence
Use connector-equivalent operations for:
- table definitions / `SHOW CREATE TABLE`;
- column metadata;
- `SHOW INDEX` / index metadata;
- constraints;
- row/cardinality statistics;
- read-only `EXPLAIN` / `EXPLAIN FORMAT=JSON` where supported;
- performance/query statistics where permission and tool support exist.

Adapt to the actual connector schema. Do not assume a specific MCP command name.

## Verification loop
For a static suspicion:
1. Identify the exact table/query relationship.
2. Read actual schema/index metadata.
3. Compare with entity/migration intent.
4. Run/read an EXPLAIN-equivalent plan when safe and supported.
5. Promote or demote confidence based on evidence.
6. Record tool/environment limitations.

## Environment handling
- Never silently switch environments.
- Never compare code against one environment and label findings as another.
- If multiple environments are available, preserve the environment identity for every DB-derived observation.
- If no DB tool is available, continue static analysis and mark DB verification unavailable; do not block the full static audit.
