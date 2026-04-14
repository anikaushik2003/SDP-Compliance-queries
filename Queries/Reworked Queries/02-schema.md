# SDP Compliance Materialized View — Final Schema

---

## Grain

**One row per stage-run-policy:** `(CorrelationId, PolicyName)`

A stage that evaluates 4 SDP policies produces 4 rows. This replaces the old design where 4 policies were pivoted into 4 hardcoded status columns on a single stage-run row.

---

## Columns

| Column | Type | Source | Purpose |
|--------|------|--------|---------|
| **Time** | | | |
| InsertedAt | datetime | GatingRequestMetrics | **Partition key for daily slicing.** Kusto ingestion time. Same value across all derived tables for a given CorrelationId (transactional update policies). |
| MetricCreationTime | datetime | GatingRequestMetrics | When the gating API ran. Differs from InsertedAt by queued ingestion delay (minutes to hours). Useful for "when did this actually happen" reporting. |
| **Identity** | | | |
| CorrelationId | string | GatingRequestMetrics | Natural key. Format: `{org}_{projectId}_{buildId}_{stage}_{attempt}` |
| AdoAccount | string | `tolower(OrganizationName)` | ADO organization |
| ProjectId | string | GatingRequestMetrics | ADO project GUID |
| DefinitionId | string | GatingRequestMetrics | Pipeline definition ID (requires Phase 3 code fix) |
| BuildId | string | GatingRequestMetrics | Run/build identifier |
| StageName | string | GatingRequestMetrics | Deployment stage name |
| PipelineUrl | string | Constructed | `https://dev.azure.com/{org}/{projectId}/_build?definitionId={defId}&_a=summary` |
| RunUrl | string | Constructed | `{PipelineUrl base}/results?buildId={BuildId}&view=results` |
| **Policy (the grain)** | | | |
| PolicyDomain | string | RuleExecutionMetrics | `SafeDeploymentPolicies` or `ChangeManagementPolicies`. SDP scope = filter on this. |
| PolicyName | string | RuleExecutionMetrics | e.g., `ring-bake-time`, `ring-progression`, `stage-bake-time`, `min-stage-count` |
| PolicyCompliantStatus | string | RuleExecutionMetrics | `Compliant`, `NotCompliant`, `CompliantWithRemediation`, or empty (V1 NotEnabled) |
| **Stage enrichment** | | | |
| Ring | string | GatingRequestMetrics | Deployment ring (e.g., TEST, SDF, MSIT, PROD) |
| Namespace | string | GatingRequestMetrics | Cosmic namespace or `"Non-Cosmic"` |
| Cloud | string | GatingRequestMetrics | Target cloud (public, dod, gcch, etc.) |
| DeploymentType | string | GatingRequestMetrics | Normal, Emergency, Hotfix, GlobalOutage |
| Trainset | string | GatingRequestMetrics | Trainset identifier |
| isCosmicStage | int | Derived from IsCosmicService | 1 if this stage is Cosmic, 0 otherwise. **Stage-level only.** |
| **Task classification** | | | |
| HasLockbox | int | Always 1 | Gating implies lockbox |
| HasClassic | int | Derived from Ev2ServiceType | 1 if RS Only or Hybrid |
| HasRA | int | Derived from Ev2ServiceType | 1 if RA Only or Hybrid |
| **Onboarding** | | | |
| Onboarded | int | Computed | 1 if Ring is non-empty AND in workload's AllowedRings |
| **YAML** | | | |
| YamlId | string | BuildYamlMapSnapshot (OneBranch) | YAML definition identifier |
| PipelineName | string | BuildYamlSnapshot (OneBranch) | Pipeline display name |
| **ServiceTree** | | | |
| ServiceId | string | GatingRequestMetrics.ServiceTreeId | ServiceTree GUID |
| Workload | string | ServiceTreeHierarchySnapshot | Computed workload grouping |
| DevOwner | string | ServiceTreeSnapshot | Dev owner |
| DivisionName | string | ServiceTreeHierarchySnapshot | Division |
| OrganizationName | string | ServiceTreeHierarchySnapshot | Organization |
| ServiceGroupName | string | ServiceTreeHierarchySnapshot | Service group |
| TeamGroupName | string | ServiceTreeHierarchySnapshot | Team group |
| ServiceName | string | ServiceTreeHierarchySnapshot | Service name |

**Primary key:** `(CorrelationId, PolicyName)`

---

---

## ServiceClassification — Dimension Table (separate)

Joined at query time via `ServiceId`. Not part of the daily slice — upserted separately.

| Column | Type | Description |
|--------|------|-------------|
| ServiceId | string | Primary key |
| TotalPipelines | long | Cumulative distinct pipeline count |
| CosmicPipelines | long | Pipelines with any Cosmic stage |
| ClassicPipelines | long | Pipelines with ExpressV2Internal stages |
| RAPipelines | long | Pipelines with Ev2RARollout stages |
| CosmicStages | long | Total Cosmic stages observed |
| ClassicStages | long | Total Classic stages observed |
| RAStages | long | Total RA stages observed |
| isCosmicService | string | `"Cosmic Service"` or `"Entirely Non-Cosmic"` |
| Ev2ServiceType | string | `"RS Only"`, `"RA Only"`, `"Hybrid Services"`, `"Unknown"` |
| LastUpdated | datetime | Last upsert time |

**Why upsert, not recompute:** Service classification is cumulative. A service that had an RA pipeline 90 days ago shouldn't lose that classification when the pipeline falls outside the rolling window. Counters only grow — recomputing from a rolling window would make classification window-dependent.

**See:** `03-service-classification.md` for table creation, initial seed, and daily upsert queries.

**PipelineOnboarded** is also a cross-row aggregate (pipeline-level, not row-level). Computed as a view over SDPMaterialized at query time — see `03-service-classification.md`.

---

## Columns Removed (vs old schema)

| Removed | Why |
|---------|-----|
| UniqueStageId | Was a join key for the old 3-table design. CorrelationId replaces it. |
| ProjectName | Caller only sends ProjectId GUID, not display name. Redundant with ProjectId. |
| RingBakeTime_Status | Replaced by per-row PolicyName + PolicyCompliantStatus |
| RingProgression_Status | Same |
| StageBakeTime_Status | Same |
| MinStageCount_Status | Same |
| Ev2ServiceType | Service-level aggregate — moved to ServiceClassification dimension table |
| isCosmicService | Service-level aggregate — moved to ServiceClassification dimension table |
| PipelineOnboarded | Pipeline-level aggregate — computed at query time from SDPMaterialized |
| HealthEnabled | Health checks run after gating — not in gating tables |
| HealthCheckPassed | Same |

---

## Time Slicing

**Time axis: `InsertedAt`** (Kusto ingestion time, not `MetricCreationTime`)

### Why InsertedAt, not MetricCreationTime

- `MetricCreationTime` = when the gating API ran (set in C# via `DateTime.UtcNow`)
- `InsertedAt` = when Kusto actually ingested the row (set by update policy via `now()`)
- These differ because queued ingestion is async — delay can be minutes to hours
- `InsertedAt` is the correct axis because it reflects when data is **available** in Kusto

### Why InsertedAt is consistent across derived tables

All 4 update policies fire on the **same RawMetrics ingestion event** with `IsTransactional: true`. This means:
- All derived tables get the **same** `InsertedAt` for the same CorrelationId
- `now()` is evaluated once within the same ingestion transaction
- If any update policy fails, the entire ingestion rolls back (all-or-nothing)
- **There is no scenario** where GatingRequestMetrics has a CorrelationId at `InsertedAt = Day 1` but RuleExecutionMetrics has it at `InsertedAt = Day 5` — they are always in the same time window

### Late-arriving data

Handled naturally. If the gating API ran on Monday but queued ingestion was delayed until Wednesday:
- All derived tables get `InsertedAt = Wednesday`
- Wednesday's slice picks up the complete row across all tables together
- The 2-day sliding window (`[D-1, D+1)`) covers delays up to ~48 hours
- Beyond ~48 hours, the row is likely dead-lettered by Kusto's ingestion queue

### Join safety

| Source | Sliced by | Why it's safe |
|--------|-----------|---------------|
| GatingRequestMetrics | `InsertedAt` | Same ingestion transaction as RuleExecutionMetrics |
| RuleExecutionMetrics | `InsertedAt` | Same ingestion transaction as GatingRequestMetrics |
| BuildYamlMapSnapshot | N/A | Scoped by day's `(PipelineUrl, BuildId)` — a BuildId belongs to one ingestion event |
| ServiceTree snapshots | N/A | Static snapshots — no time filter |

### Accumulation pattern

```
Day 1: process(d1_start, d1_end) → .set-or-append Table
Day 2: process(d2_start, d2_end) → .set-or-append Table
...
Day N: process(dN_start, dN_end) → .set-or-append Table
```

Daily sliding window (late data):
```
Each night:
  .delete from SDPMaterialized where InsertedAt between (D-1 .. D+1)
  .set-or-append SDPMaterialized <| process(D-1, D+1)
```

---

## Join Graph

```
GatingRequestMetrics          RuleExecutionMetrics
(1 row / stage-run)           (N rows / stage-run,
 filtered by                   one per PolicyDomain x PolicyName,
 MetricCreationTime)           filtered by InsertedAt + PolicyDomain)
        |                              |
        |  INNER JOIN ON CorrelationId |
        |──────────────────────────────|
        |
        v
  stage-run-policy grain
        |
        |  LEFT JOIN BuildYamlMapSnapshot + BuildYamlSnapshot
        |  ON (PipelineUrl, BuildId)
        |
        |  LEFT JOIN ServiceTree ON ServiceTreeId
        |
        |  + Derived: HasLockbox, HasClassic, HasRA, isCosmicStage, Onboarded
        v
  Final materialized row
```

---

## Scoping

**Invariant:** `PolicyDomain == "SafeDeploymentPolicies"` on RuleExecutionMetrics guarantees:
- Stage has EV2 task (ExpressV2Internal or Ev2RARollout)
- Stage has Lockbox task
- Pipeline is above-ARM MOBR

**Denominator:** Stages where SDP evaluation occurred. NOT all in-scope MOBR pipelines. Stages where the gating task didn't run are absent.

---

## Data Sources

| Source | Table | What it provides | Cluster |
|--------|-------|-----------------|---------|
| PolicyHub | m365gating_GatingRequestMetrics | Scoping + all enrichment columns | policyhub-kusto-prod |
| PolicyHub | m365gating_RuleExecutionMetrics | PolicyDomain, PolicyName, PolicyCompliantStatus | policyhub-kusto-prod |
| OneBranch | BuildYamlMapSnapshot | YamlId per (PipelineUrl, BuildId) | onebranchm365release |
| OneBranch | BuildYamlSnapshot | PipelineName per YamlId | onebranchm365release |
| OneBranch | ServiceTreeHierarchySnapshot | Workload, org hierarchy | onebranchm365release |
| OneBranch | ServiceTreeSnapshot | DevOwner | onebranchm365release |

**Eliminated:** Logs, TimelineRecords, PipelineRecords.
