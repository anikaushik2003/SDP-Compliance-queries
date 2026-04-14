# Reworked Queries — PolicyHub as Data Source

These queries replace the original 3 SDP compliance sub-queries with a single parameterized daily-slice query that sources data from PolicyHub normalized tables.

## Files

| File | Purpose |
|------|---------|
| `01-daily-slice.md` | The query. Parameterized by `(sliceStart, sliceEnd)`. Produces one day's worth of stage-run-policy rows. ADF calls this per day and appends results. |
| `02-schema.md` | Final materialized view schema, ServiceClassification dimension table, join graph, time slicing design. |
| `03-service-classification.md` | ServiceClassification dimension table: creation, initial seed, daily upsert, PipelineOnboarded computation. Separate from the daily slice — upserted after each daily append. |
| `04-dashboard-query.md` | The Power BI query. Reads from SDPMaterialized + ServiceClassification. Includes row count estimation with reasoning. |

## Data Source Shift

| Before (OneBranch) | After (PolicyHub) |
|---------------------|-------------------|
| `Logs` — 3 message types, regex + JSON parsing | `m365gating_GatingRequestMetrics` — direct column reads |
| `TimelineRecords` — task scanning, health checks | `m365gating_GatingRequestMetrics` — Ev2ServiceType column |
| `PipelineRecords` — scoping, ServiceId, Version | `m365gating_GatingRequestMetrics` — scoping by existence |
| `Logs` pivot → 4 hardcoded status columns | `m365gating_RuleExecutionMetrics` — per-row PolicyName + PolicyCompliantStatus |

**Still from OneBranch:** BuildYamlMapSnapshot/BuildYamlSnapshot (YAML bridge), ServiceTree snapshots.

## Grain

**stage-run-policy** — one row per `(CorrelationId, PolicyName)`.

Old grain was stage-run with 4 pivoted status columns. New grain has PolicyName as a first-class column — no hardcoded policy list, no summarize.

## Scoping

**Invariant:** `PolicyDomain == "SafeDeploymentPolicies"` on RuleExecutionMetrics guarantees lockbox + EV2 + above-ARM. No separate below-ARM filter or PipelineRecords scoping needed.

**Denominator:** Stages where SDP evaluation occurred — not all in-scope MOBR pipelines. Stages where gating didn't run are absent.

## Time Slicing

**Time axis: `InsertedAt`** (Kusto ingestion time), not `MetricCreationTime` (gating API time).

`InsertedAt` is guaranteed identical across `GatingRequestMetrics` and `RuleExecutionMetrics` for the same CorrelationId because all 4 update policies fire in the same `IsTransactional: true` ingestion event. There is no scenario where a CorrelationId appears in one derived table on Day 1 and another on Day 5.

Late-arriving data is handled naturally: delayed ingestion means all derived tables land together in the same later time window. The 2-day sliding window (`[D-1, D+1)` with delete-then-append) covers delays up to ~48 hours.

## Prerequisites

1. **DefinitionId code fix** — must be populated in `GatingWorkflowBase.WriteResultToKustoAsync()` (currently `string.Empty`)
2. **6 enrichment columns** — Trainset, DeploymentType, Ring, Namespace, IsCosmicService, Ev2ServiceType on GatingRequestMetrics
3. **RuleExecutionMetrics schema fix** — PolicyDomain, PolicyName, PolicyCompliantStatus columns
4. **Backfill** — historical data from RawMetrics + Logs

See [Enriching-Gating-Normalized-Tables.md](../../Documents/Enriching-Gating-Normalized-Tables.md) for full details.
