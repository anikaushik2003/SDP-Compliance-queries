# SDP Compliance Queries - Schema Reference

---

## Table 1: all-stage-runs (Central Fact Table — Scoping)

**Source file:** `all-stage-runs.md`
**Description:** All above-ARM MOBR pipeline stage-level run records (last 60 days). Defines the universe of in-scope stages. Filters out test accounts, AAD, below-ARM pipelines, and excluded ServiceTreeGuids. The `exemptedPipelines` filter must also be applied here (or as a post-join filter) to exclude known-exempt pipelines.
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
| (PipelineUrl, BuildId) | yaml-to-run-list | left outer | Add YamlId, PipelineName to each run |
| UniqueStageId | stage-telemetry-policy-compliance | left outer | Add runtime enrichment + policy statuses + ServiceTree + Onboarded |

---

## Table 2: yaml-to-run-list (YAML Bridge + Pipeline Name)

**Source file:** `yaml-to-run-list.md`
**Description:** Maps each pipeline run (BuildId) to its YAML definition (YamlId) and pipeline display name. Enrichment only — no downstream join depends on YamlId.
**Grain:** One row per run per pipeline.

| Column | Type | Description |
|---|---|---|
| PipelineUrl | string | ADO pipeline URL |
| BuildId | long | Run/build identifier |
| YamlId | string | YAML definition identifier from BuildYamlMapSnapshot |
| PipelineName | string | Pipeline display name from BuildYamlSnapshot (Index == 0) |

**Primary Key:** `(PipelineUrl, BuildId)`

**Foreign Keys:**

| FK Columns | Joins To | Join Type | Purpose |
|---|---|---|---|
| (PipelineUrl, BuildId) | all-stage-runs | left outer | Bridge: connect runs to their YAML + pipeline name |

**Relationship:** Each (PipelineUrl, BuildId) maps to exactly one YamlId. Multiple BuildIds can share the same YamlId (same YAML used across runs).

---

## Table 3: stage-telemetry-policy-compliance (Runtime Enrichment + ServiceTree)

**Source file:** `stage-telemetry-policy-compliance.md`
**Description:** Per-stage runtime data combining: policy compliance results (from Logs), health check status (from TimelineRecords), stage properties (ring, namespace, cloud, deploymentType, trainset from Logs), task classification flags (HasLockbox, HasClassic, HasRA from TimelineRecords), ServiceTree enrichment (Workload, DivisionName, etc.), and stage onboarding status (Onboarded, computed from Ring + Workload's AllowedRings).
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
| **Onboarding** | | |
| Onboarded | int (0/1) | 1 if Ring is non-empty AND Ring is in the workload's AllowedRings list; 0 otherwise |
| **ServiceTree enrichment** (via ServiceId from GetVersionInfo) | | |
| ServiceId | string | ServiceTree GUID |
| Workload | string | Computed workload grouping (from ServiceTree hierarchy) |
| DevOwner | string | Dev owner from ServiceTree |
| DivisionName | string | Division from ServiceTree hierarchy |
| OrganizationName | string | Organization from ServiceTree hierarchy |
| ServiceGroupName | string | Service group from ServiceTree hierarchy |
| TeamGroupName | string | Team group from ServiceTree hierarchy |
| ServiceName | string | Service name from ServiceTree hierarchy |

**Primary Key:** `UniqueStageId`

**Foreign Keys:**

| FK Columns | Joins To | Join Type | Purpose |
|---|---|---|---|
| UniqueStageId | all-stage-runs.UniqueStageId | left outer | Enrich scoped stage runs with all runtime + ServiceTree data |

**Data sources within this query:**
- **Logs** (3 message types): PolicyEvidenceRecord → policy statuses; MOBR API URL → Ring, Cloud, DeploymentType, Trainset; Cosmic log → CosmicRing, CosmicNamespace
- **TimelineRecords**: HealthEnabled, HealthCheckPassed, HasLockbox, HasClassic, HasRA
- **PipelineRecords** (via GetVersionInfo lookup): Version, ServiceId (for StageName derivation + ServiceTree join)
- **ServiceTreeHierarchySnapshot + ServiceTreeSnapshot**: Workload, DivisionName, OrganizationName, ServiceGroupName, TeamGroupName, ServiceName, DevOwner
- **ringworkload datatable**: AllowedRings per Workload (for Onboarded computation)

**Note:** HasLockbox, HasClassic, and HasRA are deliberately sourced from TimelineRecords (runtime task detection) rather than YAML definitions. Runtime detection is more accurate because it reflects what actually executed, not what was defined.

---

## Join Graph

```
    yaml-to-run-list
   (PipelineUrl, BuildId, YamlId, PipelineName)
            |
   JOIN ON: (PipelineUrl, BuildId)
            |
            v
    all-stage-runs ──────────────────── stage-telemetry-policy-compliance
    (PipelineUrl,                       (UniqueStageId,
     BuildId,        JOIN ON:            Ring, Namespace, Cloud,
     StageName,      UniqueStageId       DeploymentType, Trainset,
     ServiceTreeGuid,                    isCosmicStage, Onboarded,
     UniqueStageId,                      HasLockbox, HasClassic, HasRA,
     AdoAccount,                         RingBakeTime_Status,
     ProjectId,                          RingProgression_Status,
     ProjectName)                        StageBakeTime_Status,
                                         MinStageCount_Status,
                                         HealthEnabled, HealthCheckPassed,
                                         ServiceId, Workload, DevOwner,
                                         DivisionName, OrganizationName,
                                         ServiceGroupName, TeamGroupName,
                                         ServiceName)
```

---

## Assembly Order

```
Step 1:  all-stage-runs
              |
              | LEFT OUTER JOIN yaml-to-run-list ON (PipelineUrl, BuildId)
              v
         + YamlId, PipelineName per row
              |
Step 2:       | LEFT OUTER JOIN stage-telemetry-policy-compliance ON UniqueStageId
              v
         + Ring, Namespace, Cloud, DeploymentType, Trainset, isCosmicStage,
           HasLockbox, HasClassic, HasRA, Onboarded,
           RingBakeTime_Status, RingProgression_Status,
           StageBakeTime_Status, MinStageCount_Status,
           HealthEnabled, HealthCheckPassed,
           ServiceId, Workload, DevOwner, DivisionName, OrganizationName,
           ServiceGroupName, TeamGroupName, ServiceName
```

**Step 1** is left outer because YamlId and PipelineName are purely enrichment metadata — no downstream table uses them as join keys. Runs without a match in BuildYamlMapSnapshot are kept with NULL values.
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

1. **Type mismatch on BuildId:** `BuildId` in all-stage-runs is `long`; `Id` in stage-telemetry-policy-compliance is `string`. If joining on (PipelineUrl, BuildId, StageName) instead of UniqueStageId, one side needs a cast.

2. **Join type rationale:**
   - **Step 1 (left outer):** YamlId and PipelineName are purely enrichment metadata — no downstream table uses them as join keys. Runs without a YamlId are kept with NULL values.
   - **Step 2 (left outer):** Policy telemetry, health checks, ServiceTree enrichment, and Onboarded status all depend on runtime data existing. Stages where telemetry hasn't arrived yet will have NULL enrichment columns.

3. **Role separation:**
   - **all-stage-runs** = scoping ("which stages exist in the MOBR universe")
   - **yaml-to-run-list** = bridge + metadata ("which YAML and pipeline name per run")
   - **stage-telemetry-policy-compliance** = enrichment ("what happened at runtime + ServiceTree + onboarding status")

   all-stage-runs cannot be replaced by policy-compliance because: (a) policy-compliance has no MOBR/above-ARM scoping filters, (b) policy-compliance only has data for stages where telemetry exists — stages with no log entries would be lost, (c) ServiceTreeGuid (from PipelineRecords) and ServiceGroupName (from PipelineRecords) are only in all-stage-runs.

4. **Onboarded computation:** `Onboarded = iff(Ring == "", 0, iff(AllowedRings has Ring, 1, 0))`. This requires ServiceTree enrichment (ServiceId → Workload) and the ringworkload datatable (Workload → AllowedRings). Both are now computed within stage-telemetry-policy-compliance.

---

## Incremental Processing Support

All 3 tables and their joins support daily incremental processing. The full X-day result can be built by processing 1-day slices and accumulating the enriched daily outputs.

### Per-Table Partitionability

| Table | Partitionable by day? | Why |
|---|---|---|
| all-stage-runs | Yes | PipelineRecords filtered by `Timestamp between (now()-1d..now())`. Each day's slice gives that day's runs. UNION daily slices = full X days. |
| yaml-to-run-list | Yes | Each run maps to exactly one YamlId + PipelineName. Per-run, no cross-day dependency. |
| stage-telemetry-policy-compliance | Yes | Logs filtered by `Timestamp > ago(1d)`. Each record is per stage per run. ServiceTree and ringworkload are snapshot/static tables — no cross-day dependency. |

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

All other downstream enrichments (PipelineOnboarded, exempted_pipelines) are per-run or static — all fine incrementally.

---

## External Table: GatingRequestMetrics

**Cluster:** `m365gating-kusto-prod.centralus`
**Database:** `M365GatingAnalytics`
**Table:** `GatingRequestMetrics`
**Description:** Gating request metrics for M365 pipelines, capturing per-stage gating compliance status and metadata.

| # | Column | Type | Description |
|---|---|---|---|
| 0 | CorrelationId | string | Correlation identifier for the gating request |
| 1 | InsertedAt | datetime | Timestamp when the record was inserted |
| 2 | MetricCreationTime | datetime | Timestamp when the metric was created |
| 3 | OrganizationName | string | Azure DevOps organization/account name (same as AdoAccount) |
| 4 | ProjectName | string | ADO project name |
| 5 | ProjectId | string | ADO project GUID |
| 6 | DefinitionId | string | Pipeline definition ID |
| 7 | BuildId | string | Run/build identifier |
| 8 | ServiceTreeId | string | ServiceTree GUID |
| 9 | StageName | string | Deployment stage name |
| 10 | StageAttempt | string | Stage attempt number |
| 11 | JobAttempt | string | Job attempt number |
| 12 | Cloud | string | Target cloud environment |
| 13 | IsProduction | bool | Whether this is a production deployment |
| 14 | GateType | string | Type of gate applied |
| 15 | OverallGatingCompliantStatus | string | Overall gating compliance status |
| 16 | Metadata | dynamic | Additional metadata (JSON) |

**Cols to be added:**

| Column | Type | Description |
|---|---|---|
| Trainset | string | Trainset ID |
| DeploymentType | string | Deployment type |
| Ring | string | Deployment ring |
| Namespace | string | COSMIC namespace, or "Non-Cosmic" |
| isCosmicService | int (0/1) | 1 if service has COSMIC pipelines |
| Ev2ServiceType | string | EV2 service classification (e.g. "RS Only", "RA Only", "Hybrid") |

---

## External Table: RuleExecutionMetrics

**Cluster:** `m365gating-kusto-prod.centralus`
**Database:** `M365GatingAnalytics`
**Table:** `RuleExecutionMetrics`
**Description:** Per-build rule execution results, capturing whether a policy ran and passed along with associated data and metadata.

| # | Column | Type | Description |
|---|---|---|---|
| 0 | CorrelationId | string | Correlation identifier for the rule execution |
| 1 | InsertedAt | datetime | Timestamp when the record was inserted |
| 2 | BuildId | string | Run/build identifier |
| 3 | PolicyRan | bool | Whether the policy was executed |
| 4 | PolicyPassed | bool | Whether the policy passed |
| 5 | Data | string | Rule execution data |
| 6 | MetaData | dynamic | Additional metadata (JSON) |

**Cols to be added:**

| Column | Type | Description |
|---|---|---|
| PolicyName | string | Name of the policy |
| Version | string | Policy version |
| Mode | string | Policy execution mode |

---

## External Table: m365gating_GatingRequestMetrics

**Cluster:** `policyhub-kusto-prod.centralus`
**Database:** `PolicyHubAnalytics`
**Table:** `m365gating_GatingRequestMetrics`
**Description:** Follower/materialized view of GatingRequestMetrics from the M365Gating cluster, capturing per-stage gating compliance status and metadata.

| # | Column | Type | Description |
|---|---|---|---|
| 0 | CorrelationId | string | Correlation identifier for the gating request |
| 1 | MetricCreationTime | datetime | Timestamp when the metric was created |
| 2 | OrganizationName | string | Azure DevOps organization/account name (same as AdoAccount) |
| 3 | ProjectName | string | ADO project name |
| 4 | ProjectId | string | ADO project GUID |
| 5 | DefinitionId | string | Pipeline definition ID |
| 6 | BuildId | string | Run/build identifier |
| 7 | ServiceTreeId | string | ServiceTree GUID |
| 8 | StageName | string | Deployment stage name |
| 9 | StageAttempt | string | Stage attempt number |
| 10 | JobAttempt | string | Job attempt number |
| 11 | Cloud | string | Target cloud environment |
| 12 | IsProduction | bool | Whether this is a production deployment |
| 13 | GateType | string | Type of gate applied |
| 14 | OverallGatingCompliantStatus | string | Overall gating compliance status |
| 15 | Metadata | dynamic | Additional metadata (JSON) |

**Cols to be added:**

| Column | Type | Description |
|---|---|---|
| Trainset | string | Trainset ID |
| DeploymentType | string | Deployment type |
| Ring | string | Deployment ring |
| Namespace | string | COSMIC namespace, or "Non-Cosmic" |
| isCosmicService | int (0/1) | 1 if service has COSMIC pipelines |
| Ev2ServiceType | string | EV2 service classification (e.g. "RS Only", "RA Only", "Hybrid") |

---

## External Table: m365gating_RuleExecutionMetrics

**Cluster:** `policyhub-kusto-prod.centralus`
**Database:** `PolicyHubAnalytics`
**Table:** `m365gating_RuleExecutionMetrics`
**Description:** Follower/materialized view of RuleExecutionMetrics from the M365Gating cluster, capturing per-build rule execution results.

| # | Column | Type | Description |
|---|---|---|---|
| 0 | CorrelationId | string | Correlation identifier for the rule execution |
| 1 | BuildId | string | Run/build identifier |
| 2 | PolicyRan | bool | Whether the policy was executed |
| 3 | PolicyPassed | bool | Whether the policy passed |
| 4 | Data | string | Rule execution data |
| 5 | MetaData | dynamic | Additional metadata (JSON) |

**Cols to be added:**

| Column | Type | Description |
|---|---|---|
| PolicyName | string | Name of the policy |
| Version | string | Policy version |
| Mode | string | Policy execution mode |
