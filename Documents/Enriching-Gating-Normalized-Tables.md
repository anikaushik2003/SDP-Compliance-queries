# Enriching Gating Normalized (Derived) Kusto Tables

This document describes the plan to add new columns to the M365 Gating Kusto tables — specifically `RawMetrics` and the derived `RuleExecutionMetrics` table — to support SDP compliance dashboards and stage-level analytics.

---

## Table of Contents

1. [Background](#background)
2. [Current Kusto Architecture](#current-kusto-architecture)
3. [New Columns Overview](#new-columns-overview)
4. [Column Source Mapping](#column-source-mapping)
5. [Changes Required](#changes-required)
   - [KustoTables.bicep](#1-kustotablesbicep)
   - [KustoUpdatePolicies.bicep](#2-kustoupdatepoliciesbicep)
   - [GatingWorkflowBase.cs](#3-gatingworkflowbasecs)
   - [MOBRGatingRequestParams.cs](#4-mobrgatingrequestparamscs)
   - [Ev2ServiceType Derivation](#5-ev2servicetype-derivation)
6. [Deployment](#deployment)
7. [Reference Queries](#reference-queries)

---

## Background

The SDP Compliance dashboards (see `Sdp-Compliance-Queries/Queries/`) currently derive stage-level enrichment columns (Ring, Trainset, DeploymentType, Namespace, Cosmic status, EV2 service type) by joining across multiple external data sources (`TimelineRecords`, `Logs`, `BuildYamlSnapshot`, etc.) at query time. This is expensive and fragile.

Since the M365 Gating service already receives most of these values as request parameters during policy evaluation, we can capture them at ingestion time into `RawMetrics` and propagate them into the derived tables via Kusto update policies. This eliminates the need for complex joins in downstream queries.

---

## Current Kusto Architecture

### How Data Flows

```
Gating API Request
  |
  v
GatingWorkflowBase.WriteResultToKustoAsync()
  |  Serializes result + request params to JSON
  v
KustoClient.IngestAsync()  (queued ingestion)
  |
  v
RawMetrics table  (primary ingestion target)
  |  Kusto Update Policies (automatic, transactional)
  |--- GatingRequestMetrics    (projects core request fields)
  |--- PolicyDomainMetrics     (expands DomainResults array)
  |--- PolicyVersionMetrics    (expands policy version results)
  |--- RuleExecutionMetrics    (expands rule execution details)
```

### Key Files

| File | Purpose |
|------|---------|
| `M365GatingEV2Specs/Templates/KustoTables.bicep` | Table schemas + JSON ingestion mapping |
| `M365GatingEV2Specs/Templates/KustoUpdatePolicies.bicep` | Update policy definitions for derived tables |
| `M365GatingEV2Specs/Templates/KustoCluster.bicep` | Cluster + database provisioning |
| `M365GatingEV2Specs/Templates/KustoRoleAssignments.bicep` | RBAC configuration |
| `Workflows/ComplianceGating/GatingWorkflowBase.cs` | Orchestrates Kusto ingestion in `WriteResultToKustoAsync()` |
| `Models/MOBRGatingRequestParams.cs` | Request parameter model (source of most new column values) |
| `Models/Configs/KustoConfig.cs` | Cluster URL, database, table, mapping config |

### What Are Update Policies?

Update policies are an Azure Data Explorer feature that automatically transforms and routes data from one table into other tables at ingestion time. When a row lands in `RawMetrics`, Kusto runs predefined KQL functions that extract/expand nested data from the `RawGatingResult` dynamic column and insert results into derived tables — within the same ingestion transaction. No separate ETL or scheduled job is needed.

---

## New Columns Overview

### Columns to Add to `RuleExecutionMetrics`

| Column | Type | Description |
|--------|------|-------------|
| `PolicyName` | string | Name of the gating policy (e.g., `ring-bake-time`, `ring-progression`, `stage-bake-time`, `min-stage-count`) |
| `Version` | string | Policy version identifier. `"default"` for flat (V1) policy results |
| `PolicyMode` | string | Enforcement mode of the policy (`block`, `audit`, `warning`, `NotEnabled`) |
| `Trainset` | string | Serialized trainset information from the deployment pipeline |
| `DeploymentType` | string | Type of deployment (`Normal`, `Emergency`, `Hotfix`, `GlobalOutage`) |
| `Ring` | string | Deployment ring name (e.g., `TEST`, `SDF`, `MSIT`, `PROD`) |
| `Namespace` | string | Cosmic namespace. Defaults to `"Non-Cosmic"` if empty |
| `IsCosmicService` | bool | Whether the stage is a Cosmic deployment (derived from `CosmicNamespace` being non-empty) |
| `Ev2ServiceType` | string | EV2 deployment classification per stage: `"RS Only"`, `"RA Only"`, `"Hybrid Services"`, or `"Unknown"` |

### Columns to Add to `RawMetrics`

All columns except `PolicyName`, `Version`, and `PolicyMode` are added to `RawMetrics` (the ingestion target). The policy-level columns are only in `RuleExecutionMetrics` because they are extracted from the nested `RawGatingResult` via the update policy.

| Column | Type | Added to `RawMetrics`? | Added to `RuleExecutionMetrics`? |
|--------|------|:----------------------:|:--------------------------------:|
| PolicyName | string | No (nested in `RawGatingResult`) | Yes (extracted by update policy) |
| Version | string | No (nested in `RawGatingResult`) | Yes (extracted by update policy) |
| PolicyMode | string | No (nested in `RawGatingResult`) | Yes (extracted by update policy) |
| Trainset | string | Yes | Yes (projected from `RawMetrics`) |
| DeploymentType | string | Yes | Yes (projected from `RawMetrics`) |
| Ring | string | Yes | Yes (projected from `RawMetrics`) |
| Namespace | string | Yes | Yes (projected from `RawMetrics`) |
| IsCosmicService | bool | Yes | Yes (projected from `RawMetrics`) |
| Ev2ServiceType | string | Yes | Yes (projected from `RawMetrics`) |

---

## Column Source Mapping

Where each column value comes from in the gating service code:

| Column | Source Property | Source Class | Notes |
|--------|----------------|--------------|-------|
| `Trainset` | `Trainset` | `MOBRGatingRequestParams` | Direct from API request parameter |
| `DeploymentType` | `DeploymentType` | `MOBRGatingRequestParams` | `ReleaseDeploymentType` enum → string. Values: `Normal`, `Emergency`, `Hotfix`, `GlobalOutage` |
| `Ring` | `DeploymentRing` | `MOBRGatingRequestParams` | Direct from API request parameter (`ring` query param) |
| `Namespace` | `CosmicNamespace` | `MOBRGatingRequestParams` | Default to `"Non-Cosmic"` if empty (matches SDP query logic) |
| `IsCosmicService` | Derived | — | `true` if `CosmicNamespace` is non-empty |
| `Ev2ServiceType` | Derived | `ReleaseStage` | Computed from `Ev2RSJobs` / `Ev2RAJobs` on the current stage. See [derivation logic](#5-ev2servicetype-derivation) |
| `PolicyName` | `PolicyName` | `GatingPolicyResult` / `GatingPolicyResultV2` | Already in `RawGatingResult`, just not projected into `RuleExecutionMetrics` |
| `Version` | `Version` | `PolicyVersionResult` | Already in `RawGatingResult`. `"default"` for flat V1 results |
| `PolicyMode` | `PolicyMode` | `PolicyVersionResult` / `GatingPolicyResult` | Already in `RawGatingResult` |

---

## Changes Required

### 1. KustoTables.bicep

**File:** `M365GatingEV2Specs/Templates/KustoTables.bicep`

#### a) Add columns to `RawMetrics` table (after `RawGatingResult` column)

```kql
.create-merge table RawMetrics (
    MetricCreationTime: datetime,
    CorrelationId: string,
    OrganizationName: string,
    ProjectName: string,
    ProjectId: string,
    DefinitionId: string,
    BuildId: string,
    ServiceTreeId: string,
    StageName: string,
    StageAttempt: string,
    JobAttempt: string,
    Cloud: string,
    IsProduction: bool,
    GateType: string,
    OverallGatingCompliantStatus: string,
    Metadata: dynamic,
    RawGatingResult: dynamic,
    Trainset: string,            // NEW
    DeploymentType: string,      // NEW
    Ring: string,                // NEW
    Namespace: string,           // NEW
    IsCosmicService: bool,       // NEW
    Ev2ServiceType: string       // NEW
)
```

#### b) Update JSON ingestion mapping

Add to `m365gating_RawMetrics_mapping`:

```json
{"column":"Trainset","path":"$.Trainset","datatype":"string"},
{"column":"DeploymentType","path":"$.DeploymentType","datatype":"string"},
{"column":"Ring","path":"$.Ring","datatype":"string"},
{"column":"Namespace","path":"$.Namespace","datatype":"string"},
{"column":"IsCosmicService","path":"$.IsCosmicService","datatype":"bool"},
{"column":"Ev2ServiceType","path":"$.Ev2ServiceType","datatype":"string"}
```

#### c) Add columns to `RuleExecutionMetrics` table

```kql
.create-merge table RuleExecutionMetrics (
    CorrelationId: string,
    InsertedAt: datetime,
    BuildId: string,
    PolicyName: string,          // NEW
    Version: string,             // NEW
    PolicyMode: string,          // NEW
    PolicyRan: bool,
    PolicyPassed: bool,
    Data: string,
    MetaData: dynamic,
    Trainset: string,            // NEW
    DeploymentType: string,      // NEW
    Ring: string,                // NEW
    Namespace: string,           // NEW
    IsCosmicService: bool,       // NEW
    Ev2ServiceType: string       // NEW
)
```

> `.create-merge` is additive and idempotent — new columns are added without affecting existing data. Existing rows get null/default for the new columns.

---

### 2. KustoUpdatePolicies.bicep

**File:** `M365GatingEV2Specs/Templates/KustoUpdatePolicies.bicep`

Update the `RuleExecutionMetrics` update policy query. The policy has two `union` branches that both need the new projected columns.

#### Branch 1: DomainResults path (V2 response structure)

```
RawMetrics
| extend DomainResults = RawGatingResult.DomainResults
| mv-expand DomainResult = DomainResults
| where isnotnull(DomainResult)
| extend PolicyResults = DomainResult.PolicyResults
| mv-expand PolicyResult = PolicyResults
| where isnotnull(PolicyResult)
| extend PolicyVersionResults = PolicyResult.PolicyVersionResults
| mv-expand PolicyVersionResult = PolicyVersionResults
| where isnotnull(PolicyVersionResult)
| extend RuleResults = PolicyVersionResult.RuleResults
| mv-expand RuleResult = RuleResults
| where isnotnull(RuleResult)
| project
    CorrelationId,
    InsertedAt = now(),
    BuildId = tostring(RuleResult.BuildId),
    PolicyName = tostring(PolicyResult.PolicyName),           // NEW
    Version = tostring(PolicyVersionResult.Version),          // NEW
    PolicyMode = tostring(PolicyVersionResult.PolicyMode),    // NEW
    PolicyRan = tobool(RuleResult.PolicyRan),
    PolicyPassed = tobool(RuleResult.PolicyPassed),
    Data = tostring(RuleResult.Data),
    MetaData = RuleResult.MetaData,
    Trainset,                                                 // NEW (from RawMetrics row)
    DeploymentType,                                           // NEW
    Ring,                                                     // NEW
    Namespace,                                                // NEW
    IsCosmicService,                                          // NEW
    Ev2ServiceType                                            // NEW
```

#### Branch 2: Flat PolicyResults path (V1 response structure)

```
RawMetrics
| extend PolicyResults = RawGatingResult.PolicyResults
| mv-expand PolicyResult = PolicyResults
| where isnotnull(PolicyResult)
| extend RuleResults = PolicyResult.RuleResults
| mv-expand RuleResult = RuleResults
| where isnotnull(RuleResult)
| project
    CorrelationId,
    InsertedAt = now(),
    BuildId = tostring(RuleResult.BuildId),
    PolicyName = tostring(PolicyResult.PolicyName),           // NEW
    Version = "default",                                      // NEW (no version nesting in V1)
    PolicyMode = tostring(PolicyResult.PolicyMode),           // NEW
    PolicyRan = tobool(RuleResult.PolicyRan),
    PolicyPassed = tobool(RuleResult.PolicyPassed),
    Data = tostring(RuleResult.Data),
    MetaData = RuleResult.MetaData,
    Trainset,                                                 // NEW
    DeploymentType,                                           // NEW
    Ring,                                                     // NEW
    Namespace,                                                // NEW
    IsCosmicService,                                          // NEW
    Ev2ServiceType                                            // NEW
```

> `Trainset`, `DeploymentType`, `Ring`, `Namespace`, `IsCosmicService`, and `Ev2ServiceType` are top-level columns on `RawMetrics` and can be projected directly without extraction from `RawGatingResult`.

> `PolicyName`, `Version`, and `PolicyMode` are extracted from the nested `RawGatingResult` JSON structure, following the same pattern used by the existing `PolicyVersionMetrics` update policy.

---

### 3. GatingWorkflowBase.cs

**File:** `Workflows/ComplianceGating/GatingWorkflowBase.cs`

In `WriteResultToKustoAsync()`, add the new fields to the `rawMetricsData` anonymous object (around line 147):

```csharp
var rawMetricsData = new
{
    MetricCreationTime = DateTime.UtcNow,
    CorrelationId = correlationId,
    OrganizationName = gatingRequestParams.Organization ?? string.Empty,
    ProjectName = string.Empty,
    ProjectId = gatingRequestParams.Project ?? string.Empty,
    DefinitionId = string.Empty,
    BuildId = gatingRequestParams.BuildId ?? string.Empty,
    ServiceTreeId = gatingRequestParams.ServiceTreeId ?? string.Empty,
    StageName = gatingRequestParams.Environment ?? string.Empty,
    StageAttempt = string.Empty,
    JobAttempt = string.Empty,
    Cloud = gatingRequestParams.Cloud ?? string.Empty,
    IsProduction = gatingRequestParams.IsProduction,
    GateType = gateType.ToString(),
    OverallGatingCompliantStatus = result.CompliantStatus.ToString(),
    Metadata = parsedMetadata,
    RawGatingResult = resultForKusto,
    // --- New enrichment columns ---
    Trainset = gatingRequestParams.Trainset ?? string.Empty,
    DeploymentType = gatingRequestParams.DeploymentType.ToString(),
    Ring = gatingRequestParams.DeploymentRing ?? string.Empty,
    Namespace = string.IsNullOrEmpty(gatingRequestParams.CosmicNamespace)
        ? "Non-Cosmic"
        : gatingRequestParams.CosmicNamespace,
    IsCosmicService = !string.IsNullOrEmpty(gatingRequestParams.CosmicNamespace),
    Ev2ServiceType = gatingRequestParams.Ev2ServiceType ?? "Unknown"
};
```

---

### 4. MOBRGatingRequestParams.cs

**File:** `Models/MOBRGatingRequestParams.cs`

Add the `Ev2ServiceType` property. The other columns (`Trainset`, `DeploymentType`, `DeploymentRing`, `CosmicNamespace`) already exist on this class.

```csharp
/// <summary>
/// EV2 deployment classification for the current stage.
/// Values: "RS Only", "RA Only", "Hybrid Services", "Unknown".
/// Derived from the presence of Ev2RSJobs and Ev2RAJobs on the stage.
/// </summary>
public string Ev2ServiceType { get; set; } = "Unknown";
```

#### Existing properties used (no changes needed)

| Property | Type | Default | Maps to Kusto Column |
|----------|------|---------|---------------------|
| `Trainset` | `string` | `null` | `Trainset` |
| `DeploymentType` | `ReleaseDeploymentType` (enum) | `Normal` | `DeploymentType` |
| `DeploymentRing` | `string` | `null` | `Ring` |
| `CosmicNamespace` | `string` | `string.Empty` | `Namespace` / `IsCosmicService` |

---

### 5. Ev2ServiceType Derivation

#### Existing building blocks

The codebase already distinguishes RS and RA jobs at the stage level:

| Class / Method | Location | What it does |
|----------------|----------|--------------|
| `ReleaseStage.Ev2RSJobs` | `Models/PipelineObjects/SDP/ReleaseStage.cs` | List of Classic/RS EV2 jobs in a stage |
| `ReleaseStage.Ev2RAJobs` | `Models/PipelineObjects/SDP/ReleaseStage.cs` | List of RA EV2 jobs in a stage |
| `CommonPolicyEvaluator.IsRAOnlyAcrossDefinitions()` | `GatingPolicy/PolicyRule/SDP/CommonPolicyEvaluator.cs` | Checks if all stages across definitions have only RA jobs |
| `JobUtils.IsMobrEv2ReleaseJob()` | `Helpers/YamlUtils/JobUtils.cs` | Detects EV2 release jobs via `OneESPT.Workflow` variable or task name |
| `YamlHelper.GetStageDetails()` | `Helpers/YamlUtils/YamlHelper.cs` | Parses YAML to build `StageDetails` with RS/RA job lists |

#### Classification logic

Matches the SDP dashboard query logic (`Current state of SDP dashboard query.md`, lines 226-236):

```csharp
bool hasRS = stage.Ev2RSJobs?.Any() == true;
bool hasRA = stage.Ev2RAJobs?.Any() == true;

string ev2ServiceType = (hasRS, hasRA) switch
{
    (true, false)  => "RS Only",
    (false, true)  => "RA Only",
    (true, true)   => "Hybrid Services",
    (false, false) => "Unknown"
};
```

| `hasRS` | `hasRA` | `Ev2ServiceType` |
|:-------:|:-------:|-------------------|
| true | false | `"RS Only"` |
| false | true | `"RA Only"` |
| true | true | `"Hybrid Services"` |
| false | false | `"Unknown"` |

#### Propagation path

`Ev2ServiceType` must be computed where `StageDetails` / `ReleaseStage` data is first available, then set on `MOBRGatingRequestParams` so it reaches `WriteResultToKustoAsync()`.

```
YamlHelper.GetStageDetails()           <-- StageDetails with RS/RA jobs parsed
    |
    v
SDPUtils (policy evaluation layer)     <-- ReleaseStage accessible here
    |
    v  Compute Ev2ServiceType, set on MOBRGatingRequestParams.Ev2ServiceType
    |
MOBRGatingRequestParams                <-- Carries value through the pipeline
    |
    v
GatingWorkflowBase.WriteResultToKustoAsync()
    |
    v
RawMetrics (Kusto)
    |  Update Policy
    v
RuleExecutionMetrics (Kusto)
```

The exact insertion point should be where `SDPUtils.GetOrCreateDefinitionMetadata()` or `SDPUtils.CreateReleaseMetadata()` processes the `StageDetails` for the current request, as both methods have access to the `ReleaseStage` objects and the `Build` context.

---

## Deployment

Changes deploy via **EV2 (Geneva Safe Deployment)** through the standard pipeline:

1. Edit the Bicep templates and C# code as described above
2. PR validation runs via `.pipelines/M365Gating_PR_Validation.yml`
3. Official build compiles Bicep to ARM JSON
4. EV2 rolls out through stages:

```
Test --> PPE Canary --> PPE SDF --> PPE Production --> Prod Canary --> Prod SDF --> Prod Production
```

- `.create-merge` table commands are **additive and idempotent** — existing data is unaffected
- New columns will be `null` for historical rows
- Update policy changes take effect immediately for new ingestions after deployment

---

## Reference Queries

The following SDP compliance queries motivated these enrichments:

| Query File | What it computes | Columns it needs |
|------------|-----------------|------------------|
| `Stage-telemetry-Policy-Compliance.md` | Per-stage policy compliance with ring/cosmic/deployment enrichment | `Ring`, `Namespace`, `DeploymentType`, `Trainset`, `isCosmicStage`, `HasRA`, `HasClassic` |
| `Current state of SDP dashboard query.md` | Service-level onboarding status and EV2 classification | `Ev2ServiceType`, `isCosmicService`, `Ring`, `Namespace`, `Trainset`, `deploymentType` |

By capturing these columns at ingestion time, downstream queries can read directly from `RuleExecutionMetrics` without expensive joins against `TimelineRecords`, `Logs`, `BuildYamlSnapshot`, and `ServiceTreeHierarchySnapshot`.
