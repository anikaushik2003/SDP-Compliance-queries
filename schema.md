# SDP Compliance Queries - Schema Reference

---

## Table 1: all-stage-runs (Central Fact Table)

**Source file:** `all-stage-runs.md`
**Description:** All above-ARM MOBR pipeline stage-level run records (last 60 days). This is the central table that all other tables join into.
**Grain:** One row per stage per run per pipeline.

| Column | Type | Description |
|---|---|---|
| PipelineUrl | string | ADO pipeline URL (constructed from AdoAccount/ProjectId/DefinitionId) |
| BuildId | long | Run/build identifier (renamed from `Id`) |
| ServiceTreeGuid | string | Effective ServiceId for this stage in this run |
| StageName | string | Deployment stage name (derived from Environment) |
| ServiceGroupName | string | Service group from PipelineRecords |
| UniqueStageId | string | Composite key: `tolower(AdoAccount)\|tolower(ProjectName)\|DefinitionId\|BuildId\|StageName` |
| AdoAccount | string | Azure DevOps account name |
| ProjectId | string | ADO project GUID |
| ProjectName | string | ADO project display name |

**Primary Key:** `(PipelineUrl, BuildId, StageName)`
Equivalently: `UniqueStageId`

**Foreign Keys:**

| FK Columns | Joins To | Join Type | Purpose |
|---|---|---|---|
| (PipelineUrl, BuildId) | yaml-to-run-list | inner | Add YamlId to each run |
| (PipelineUrl, BuildId, StageName) | stage-telemetry-policy-compliance | left outer | Add policy compliance statuses |

**Note:** After joining to yaml-to-run-list (which adds YamlId), the enriched result can then join to stageDataQuery on (PipelineUrl, YamlId, StageName).

---

## Table 2: yaml-to-run-list (YAML Bridge Table)

**Source file:** `yaml-to-run-list.md`
**Description:** Maps each pipeline run (BuildId) to its YAML definition (YamlId). One YAML per build.
**Grain:** One row per run per pipeline.

| Column | Type | Description |
|---|---|---|
| PipelineUrl | string | ADO pipeline URL |
| BuildId | long | Run/build identifier |
| YamlId | string | YAML definition identifier from BuildYamlMapSnapshot |

**Primary Key:** `(PipelineUrl, BuildId)`

**Foreign Keys:**

| FK Columns | Joins To | Join Type | Purpose |
|---|---|---|---|
| (PipelineUrl, BuildId) | all-stage-runs | inner | Bridge: connect runs to their YAML |

**Relationship:** Each (PipelineUrl, BuildId) maps to exactly one YamlId. Multiple BuildIds can share the same YamlId (same YAML used across runs).

---

## Table 3: stageDataQuery (Stage YAML Definitions)

**Source file:** `stageDataQuery.md`
**Description:** Stage-level properties extracted from YAML definitions - ring, namespace, cloud, and task classification flags. These are YAML-time properties (not run-time).
**Grain:** One row per stage per YAML definition.

| Column | Type | Description |
|---|---|---|
| StageName | string | Stage name from YAML |
| PipelineUrl | string | ADO pipeline URL (derived from YamlId's Org/Project/Definition) |
| YamlId | string | YAML definition identifier |
| Ring | string | Deployment ring (from deploymentRing or COSMIC namespaceRingJson), uppercased. Empty if unresolvable. |
| Namespace | string | COSMIC namespace, or "Non-Cosmic" if not COSMIC |
| HasLockbox | int (0/1) | Stage has lockbox-approval-request-prod_with_onebranch task |
| HasClassic | int (0/1) | Stage has ExpressV2Internal (Classic EV2) task |
| HasRA | int (0/1) | Stage has Ev2RARollout (RA EV2) task |
| isCosmicStage | int (0/1) | 1 if COSMIC ring data exists, 0 otherwise |
| cloud | string | Target cloud environment (e.g., Public, GCC, GCCH) |

**Primary Key:** `(PipelineUrl, YamlId, StageName)`

**Foreign Keys:**

| FK Columns | Joins To | Join Type | Purpose |
|---|---|---|---|
| (PipelineUrl, YamlId, StageName) | all-stage-runs + yaml-to-run-list (enriched) | inner | Add stage YAML properties to run data |

**Join path:** This table does NOT join directly to all-stage-runs. The join requires YamlId, which is obtained by first joining all-stage-runs to yaml-to-run-list. The full join key is (PipelineUrl, YamlId, StageName).

---

## Table 4: stage-telemetry-policy-compliance (Policy Compliance)

**Source file:** `stage-telemetry-policy-compliance.md`
**Description:** Per-stage SDP policy evaluation results and health check status from the Logs and TimelineRecords tables.
**Grain:** One row per stage per run per pipeline.

| Column | Type | Description |
|---|---|---|
| AdoAccount | string | Azure DevOps account (lowercased) |
| ProjectName | string | ADO project name (lowercased) |
| DefinitionId | string | Pipeline definition ID |
| Id | string | Run/build identifier (note: string, not long) |
| Environment | string | Raw environment/stage identifier from ADO |
| PipelineUrl | string | ADO pipeline URL |
| RingBakeTime_Status | int | Policy status: 1=Pass/NotEnabled, 2=Fail, 3=NotRun, 4=Missing |
| RingProgression_Status | int | Policy status: 1=Pass/NotEnabled, 2=Fail, 3=NotRun, 4=Missing |
| StageBakeTime_Status | int | Policy status: 1=Pass/NotEnabled, 2=Fail, 3=NotRun, 4=Missing |
| MinStageCount_Status | int | Policy status: 1=Pass/NotEnabled, 2=Fail, 3=NotRun, 4=Missing |
| HealthEnabled | int (0/1) | 1 if health-check-v1 task exists in the stage |
| HealthCheckPassed | bool | true if all health checks passed |
| UniqueStageId | string | Composite key: `tolower(AdoAccount)\|tolower(ProjectName)\|DefinitionId\|Id\|StageName` |

**Primary Key:** `UniqueStageId`
Equivalent to: `(PipelineUrl, BuildId, StageName)` (if StageName were projected)

**Foreign Keys:**

| FK Columns | Joins To | Join Type | Purpose |
|---|---|---|---|
| UniqueStageId | all-stage-runs.UniqueStageId | left outer | Add policy statuses to stage runs |

**Type mismatch note:** `Id` in this table is `string`; `BuildId` in all-stage-runs is `long`. If joining on (PipelineUrl, BuildId, StageName) instead of UniqueStageId, cast `Id` to long or `BuildId` to string.

**Missing projection note:** `StageName` is computed internally (line 89-93) but not included in the final `project`. To join via (PipelineUrl, BuildId, StageName), add StageName to the output projection.

---

## Join Graph

```
                    yaml-to-run-list
                   (PipelineUrl, BuildId, YamlId)
                            |
                   JOIN ON: (PipelineUrl, BuildId)
                            |
                            v
    all-stage-runs  ----[enriched with YamlId]----  stageDataQuery
    (PipelineUrl,           |                       (PipelineUrl, YamlId,
     BuildId,      JOIN ON: (PipelineUrl,             StageName, Ring,
     StageName,              YamlId, StageName)        Namespace,
     StageName,             |                         cloud, HasLockbox,
     ServiceTreeGuid,       |                         HasClassic, HasRA,
     UniqueStageId,         |                         isCosmicStage)
     AdoAccount,            |
     ProjectId,             |
     ProjectName)           |
            |               |
            |    JOIN ON: UniqueStageId
            |    or (PipelineUrl, BuildId, StageName)
            v
    stage-telemetry-policy-compliance
    (UniqueStageId, RingBakeTime_Status,
     RingProgression_Status, StageBakeTime_Status,
     MinStageCount_Status, HealthEnabled,
     HealthCheckPassed)
```

---

## Assembly Order

```
Step 1:  all-stage-runs
              |
              | INNER JOIN yaml-to-run-list ON (PipelineUrl, BuildId)
              v
         + YamlId per row
              |
Step 2:       | INNER JOIN stageDataQuery ON (PipelineUrl, YamlId, StageName)
              v
         + Ring, Namespace, cloud, HasLockbox, HasClassic, HasRA, isCosmicStage
              |
Step 3:       | LEFT OUTER JOIN stage-telemetry-policy-compliance ON UniqueStageId
              v
         + RingBakeTime_Status, RingProgression_Status,
           StageBakeTime_Status, MinStageCount_Status,
           HealthEnabled, HealthCheckPassed
```

**Step 1** is inner because every run must have a YAML.
**Step 2** is inner because we only want stages that exist in the current YAML definition (filters out leftover stages from old YAMLs).
**Step 3** is left outer because not all stages have policy telemetry (policy may not have run yet).

---

## Cardinality Summary

| Join | Left | Right | Cardinality | Effect |
|---|---|---|---|---|
| all-stage-runs + yaml-to-run-list | ~564K rows | ~6K rows | Many:1 (many stages share one BuildId's YAML) | No row expansion |
| (enriched) + stageDataQuery | ~564K rows | ~583K rows | 1:1 (each run-stage matches one YAML-stage) | Filters to only YAML-defined stages |
| (enriched) + policy-compliance | result | ~558K rows | 1:1 (one policy record per stage per run) | No row expansion, NULLs for missing |

---

## Policy Status Codes Reference

| Code | Meaning |
|---|---|
| 1 | Pass (policy passed or NotEnabled) |
| 2 | Fail (policy ran and failed) |
| 3 | Not Run (policy did not execute) |
| 4 | Missing (no evidence record for this policy) |

---

## Notes and Callouts

1. **Type mismatch on BuildId:** `BuildId` in all-stage-runs is `long`; `Id` in stage-telemetry-policy-compliance is `string` (cast on line 71). If joining on (PipelineUrl, BuildId, StageName) instead of UniqueStageId, one side needs a cast (`tolong(Id)` or `tostring(BuildId)`).

2. **StageName not projected in policy-compliance:** `StageName` is computed internally (lines 89-93) but not included in the final `project` statement. Only `UniqueStageId` (which embeds StageName) is output. To join via (PipelineUrl, BuildId, StageName), add `StageName` to the output projection.

3. **Join type rationale:**
   - **Step 1 (inner):** A run with no YamlId in BuildYamlMapSnapshot cannot be enriched downstream — no YAML means no stage definitions, ring data, or task flags. Drop it.
   - **Step 2 (inner):** Runtime PipelineRecords can contain leftover stages from older YAML versions. The inner join against stageDataQuery (current YAML only) filters these out.
   - **Step 3 (left outer):** Policy telemetry depends on the SDP policy task emitting a `[PolicyEvidenceRecord]` log entry. Stages where this task hasn't run or was skipped will have no match — keep the row, leave policy columns NULL.

---

## Incremental Processing Support

All 4 tables and their joins support daily incremental processing. The full X-day result can be built by processing 1-day slices and UNIONing the enriched daily outputs.

### Per-Table Partitionability

| Table | Partitionable by day? | Why |
|---|---|---|
| all-stage-runs | Yes | PipelineRecords filtered by `Timestamp between (now()-1d..now())`. Each day's slice gives that day's runs. UNION daily slices = full X days. |
| yaml-to-run-list | Yes | Each run maps to exactly one YamlId. Per-run, no cross-day dependency. |
| stageDataQuery | Better than partitionable | Keyed on YamlId, not time. A YamlId's stage definitions are **immutable** — same YamlId always produces the same rows. Compute once per YamlId and cache. On subsequent days, only process newly discovered YamlIds. |
| stage-telemetry-policy-compliance | Yes | Logs filtered by `Timestamp > ago(1d)`. Each policy evidence record is per stage per run. No cross-day dependency. |

### Why stageDataQuery Is Cacheable

stageDataQuery's input is a set of YamlIds. A YamlId is a **snapshot of a pipeline's YAML file at a specific point in time** — once ingested into BuildYamlSnapshot, its contents never change. The query walks the YAML tree structure (stage → job → task via Index/ParentIndex) to extract Ring, Namespace, cloud, and task flags. None of these depend on when or how many times the pipeline ran. They are purely properties of the YAML definition.

- Day 1 runs produce YamlIds `{X, Y, Z}` via yaml-to-run-list
- Day 2 runs produce YamlIds `{Y, Z, W}` via yaml-to-run-list
- stageDataQuery for YamlId `Y` returns identical rows whether processed on day 1 or day 2

Two strategies for incremental processing:
1. **Deduplicate first:** Collect all YamlIds across all daily slices, deduplicate, run stageDataQuery once against the distinct set.
2. **Process per day, cache:** On day 1 process `{X, Y, Z}`. On day 2, Y and Z are cached, only process `{W}`.

### Why Joins Work Per-Day

Every join key is scoped to a single run — no cross-day dependencies:

- `(PipelineUrl, BuildId)` — a BuildId belongs to exactly one day. Day 1's runs never match day 3's yaml-to-run-list entries.
- `(PipelineUrl, YamlId, StageName)` — the YamlId came from that day's BuildId. stageDataQuery returns the same result regardless of which day you process it.
- `UniqueStageId` — encodes AdoAccount, ProjectName, DefinitionId, BuildId, StageName. Fully identifies one stage in one run on one day.

A run that happened on Tuesday never needs data from Wednesday's slice to complete its row. Each day's slice is self-contained through all 3 joins.

### Incremental Assembly Flow

Two equivalent strategies:

**Option A — Bulk UNION:**
```
Day 1:  process(day1) → Result₁
Day 2:  process(day2) → Result₂
...
Day X:  process(dayX) → Resultₓ

Final = UNION(Result₁, Result₂, ..., Resultₓ)
```

**Option B — Iterative Accumulation (preferred):**
```
Day 1:  process(day1) → Result₁ → Table = Result₁
Day 2:  process(day2) → Result₂ → Table = Table ∪ Result₂
Day 3:  process(day3) → Result₃ → Table = Table ∪ Result₃
...
Day X:  process(dayX) → Resultₓ → Table = Table ∪ Resultₓ
```

Where each `process(dayN)` is:
```
all-stage-runs(1d) → join yaml-to-run-list(1d) → join stageData → join policy(1d) → ResultN
```

Both produce the same output. Option B is preferred because:
- No need to hold all X days of raw data in memory at once
- Each day's append is a small write (1 day's worth of rows)
- If day 5 fails, days 1-4 are already persisted in the table
- Matches ADF pipeline scheduling — daily trigger, append to sink table

This works because every row is self-contained within its day. Result₃ does not depend on whether Result₁ and Result₂ are already in the table. The accumulation order does not matter.

### What Cannot Be Done Incrementally

**Service-level classification (isCosmicService, Ev2ServiceType)** — not yet modularized (still in query.md) — aggregates across ALL pipelines for a ServiceId over the full time window:

```
CosmicPipelines = dcountif(PipelineUrl, CosmicStages > 0)   // needs ALL pipelines for this ServiceId
Ev2ServiceType  = iff(TotalRAStages == 0 ..., "RS Only", ...) // needs ALL stages across all days
```

On day 1 you might see 2 of a service's 5 pipelines and classify it as "RS Only". On day 3 an RA pipeline appears and it should be "Hybrid". This requires a **recompute pass** over the full UNION'd result after all daily slices are collected.

**belowarmpipelines exclusion** — scans ALL TimelineRecords (no time filter) to build a PipelineUrl exclusion set. This is a stable pipeline-level property (pipelines don't flip between above/below ARM day to day), so compute it once and reuse across all daily slices. Minor concern only.

All other downstream enrichments (PipelineOnboarded, Onboarded, exempted_pipelines, ServiceTree lookup) are per-run, per-YAML, or static — all fine incrementally.
