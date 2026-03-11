# SDP Compliance Queries - Schema Reference

---

## Table 1: all-stage-runs (Central Fact Table — Scoping)

**Source file:** `all-stage-runs.md`
**Description:** All above-ARM MOBR pipeline stage-level run records (last 60 days). Defines the universe of in-scope stages. Filters out test accounts, AAD, below-ARM pipelines, and excluded ServiceTreeGuids.
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
| (PipelineUrl, BuildId) | yaml-to-run-list | left outer | Add YamlId to each run |
| UniqueStageId | stage-telemetry-policy-compliance | left outer | Add runtime enrichment + policy statuses |

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
| (PipelineUrl, BuildId) | all-stage-runs | left outer | Bridge: connect runs to their YAML |

**Relationship:** Each (PipelineUrl, BuildId) maps to exactly one YamlId. Multiple BuildIds can share the same YamlId (same YAML used across runs).

---

## Table 3: stage-telemetry-policy-compliance (Runtime Enrichment)

**Source file:** `stage-telemetry-policy-compliance.md`
**Description:** Per-stage runtime data combining policy compliance results (from Logs), health check status (from TimelineRecords), stage properties (ring, namespace, cloud, deploymentType, trainset from Logs), and task classification flags (HasLockbox, HasClassic, HasRA from TimelineRecords).
**Grain:** One row per stage per run per pipeline.

| Column | Type | Description |
|---|---|---|
| AdoAccount | string | Azure DevOps account (lowercased) |
| ProjectName | string | ADO project name (lowercased) |
| DefinitionId | string | Pipeline definition ID |
| Id | string | Run/build identifier (note: string, not long) |
| Environment | string | Raw environment/stage identifier from ADO |
| PipelineUrl | string | ADO pipeline URL |
| StageName | string | Deployment stage name (derived from Environment) |
| UniqueStageId | string | Composite key: `tolower(AdoAccount)\|tolower(ProjectName)\|DefinitionId\|Id\|StageName` |
| **Policy statuses** | | |
| RingBakeTime_Status | int | 1=Pass/NotEnabled, 2=Fail, 3=NotRun, 4=Missing |
| RingProgression_Status | int | 1=Pass/NotEnabled, 2=Fail, 3=NotRun, 4=Missing |
| StageBakeTime_Status | int | 1=Pass/NotEnabled, 2=Fail, 3=NotRun, 4=Missing |
| MinStageCount_Status | int | 1=Pass/NotEnabled, 2=Fail, 3=NotRun, 4=Missing |
| **Health checks** | | |
| HealthEnabled | int (0/1) | 1 if health-check-v1 task exists in the stage |
| HealthCheckPassed | bool | true if all health checks passed |
| **Stage properties** (from MOBR + Cosmic logs) | | |
| Ring | string | Deployment ring (from MOBR URL param `ring=` or Cosmic log), uppercased. Empty if unresolvable. |
| Namespace | string | COSMIC namespace, or "Non-Cosmic" if not COSMIC |
| Cloud | string | Target cloud environment (from MOBR URL param `cloud=`) |
| DeploymentType | string | Deployment type (from MOBR URL param `deploymentType=`) |
| Trainset | string | Trainset ID (from MOBR URL param `trainset=`) |
| isCosmicStage | int (0/1) | 1 if COSMIC ring data exists in logs, 0 otherwise |
| **Task classification** (from TimelineRecords) | | |
| HasLockbox | int (0/1) | Stage has lockbox-approval-request-prod_with_onebranch task |
| HasClassic | int (0/1) | Stage has ExpressV2Internal (Classic EV2) task |
| HasRA | int (0/1) | Stage has Ev2RARollout (RA EV2) task |

**Primary Key:** `UniqueStageId`

**Foreign Keys:**

| FK Columns | Joins To | Join Type | Purpose |
|---|---|---|---|
| UniqueStageId | all-stage-runs.UniqueStageId | left outer | Enrich scoped stage runs with all runtime data |

**Data sources within this query:**
- **Logs** (3 message types): PolicyEvidenceRecord → policy statuses; MOBR API URL → Ring, Cloud, DeploymentType, Trainset; Cosmic log → CosmicRing, CosmicNamespace
- **TimelineRecords**: HealthEnabled, HealthCheckPassed, HasLockbox, HasClassic, HasRA
- **PipelineRecords** (via GetVersionInfo lookup): Version, ServiceId (for StageName derivation)

---

## Join Graph

```
    yaml-to-run-list
   (PipelineUrl, BuildId, YamlId)
            |
   JOIN ON: (PipelineUrl, BuildId)
            |
            v
    all-stage-runs ──────────────────── stage-telemetry-policy-compliance
    (PipelineUrl,                       (UniqueStageId,
     BuildId,        JOIN ON:            Ring, Namespace, Cloud,
     StageName,      UniqueStageId       DeploymentType, Trainset,
     ServiceTreeGuid,                    isCosmicStage,
     UniqueStageId,                      HasLockbox, HasClassic, HasRA,
     AdoAccount,                         RingBakeTime_Status,
     ProjectId,                          RingProgression_Status,
     ProjectName)                        StageBakeTime_Status,
                                         MinStageCount_Status,
                                         HealthEnabled, HealthCheckPassed)
```

---

## Assembly Order

```
Step 1:  all-stage-runs
              |
              | LEFT OUTER JOIN yaml-to-run-list ON (PipelineUrl, BuildId)
              v
         + YamlId per row
              |
Step 2:       | LEFT OUTER JOIN stage-telemetry-policy-compliance ON UniqueStageId
              v
         + Ring, Namespace, Cloud, DeploymentType, Trainset, isCosmicStage,
           HasLockbox, HasClassic, HasRA,
           RingBakeTime_Status, RingProgression_Status,
           StageBakeTime_Status, MinStageCount_Status,
           HealthEnabled, HealthCheckPassed
```

**Step 1** is left outer because YamlId is now purely enrichment metadata — it is not a join key for any downstream table. Runs without a YamlId in BuildYamlMapSnapshot are kept with a NULL YamlId.
**Step 2** is left outer because not all stages have runtime telemetry — the SDP policy task may not have run, or log entries may be missing. Keep the stage row, leave enrichment columns NULL.

---

## Cardinality Summary

| Join | Left | Right | Cardinality | Effect |
|---|---|---|---|---|
| all-stage-runs + yaml-to-run-list | ~564K rows | ~6K rows | Many:1 (many stages share one BuildId's YAML) | No row expansion, NULLs for missing YamlId |
| (enriched) + policy-compliance | result | ~558K rows | 1:1 (one record per stage per run) | No row expansion, NULLs for missing |

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

1. **Type mismatch on BuildId:** `BuildId` in all-stage-runs is `long`; `Id` in stage-telemetry-policy-compliance is `string` (cast on line 92). If joining on (PipelineUrl, BuildId, StageName) instead of UniqueStageId, one side needs a cast (`tolong(Id)` or `tostring(BuildId)`).

2. **Join type rationale:**
   - **Step 1 (left outer):** YamlId is purely enrichment metadata — no downstream table uses it as a join key. Runs without a YamlId are kept with NULL YamlId.
   - **Step 2 (left outer):** Policy telemetry depends on the SDP policy task emitting a `[PolicyEvidenceRecord]` log entry, and runtime task data depends on TimelineRecords. Stages where these haven't run or were skipped will have no match — keep the row, leave enrichment columns NULL.

3. **Role separation:**
   - **all-stage-runs** = scoping ("which stages exist in the MOBR universe")
   - **yaml-to-run-list** = bridge ("which YAML goes with which run")
   - **stage-telemetry-policy-compliance** = enrichment ("what happened at runtime")

   all-stage-runs cannot be replaced by policy-compliance because: (a) policy-compliance has no MOBR/above-ARM scoping filters, (b) policy-compliance only has data for stages where telemetry exists — stages with no log entries would be lost, (c) ServiceTreeGuid and ServiceGroupName are only in all-stage-runs.

---

## Incremental Processing Support

All 3 tables and their joins support daily incremental processing. The full X-day result can be built by processing 1-day slices and accumulating the enriched daily outputs.

### Per-Table Partitionability

| Table | Partitionable by day? | Why |
|---|---|---|
| all-stage-runs | Yes | PipelineRecords filtered by `Timestamp between (now()-1d..now())`. Each day's slice gives that day's runs. UNION daily slices = full X days. |
| yaml-to-run-list | Yes | Each run maps to exactly one YamlId. Per-run, no cross-day dependency. |
| stage-telemetry-policy-compliance | Yes | Logs filtered by `Timestamp > ago(1d)`. Each record is per stage per run. No cross-day dependency. |

### Why Joins Work Per-Day

Every join key is scoped to a single run — no cross-day dependencies:

- `(PipelineUrl, BuildId)` — a BuildId belongs to exactly one day. Day 1's runs never match day 3's yaml-to-run-list entries.
- `UniqueStageId` — encodes AdoAccount, ProjectName, DefinitionId, BuildId, StageName. Fully identifies one stage in one run on one day.

A run that happened on Tuesday never needs data from Wednesday's slice to complete its row. Each day's slice is self-contained through both joins.

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
all-stage-runs(1d) → join yaml-to-run-list(1d) → join policy-compliance(1d) → ResultN
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

On day 1 you might see 2 of a service's 5 pipelines and classify it as "RS Only". On day 3 an RA pipeline appears and it should be "Hybrid". This requires a **recompute pass** over the full accumulated table after all daily slices are collected.

**belowarmpipelines exclusion** — scans ALL TimelineRecords (no time filter) to build a PipelineUrl exclusion set. This is a stable pipeline-level property (pipelines don't flip between above/below ARM day to day), so compute it once and reuse across all daily slices. Minor concern only.

All other downstream enrichments (PipelineOnboarded, Onboarded, exempted_pipelines, ServiceTree lookup) are per-run, per-YAML, or static — all fine incrementally.
