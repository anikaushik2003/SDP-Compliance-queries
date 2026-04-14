# ADF Backfill Pipelines — Gating Normalized Tables

This document contains the ADF pipeline definitions and KQL scripts for backfilling the M365 Gating normalized Kusto tables. It covers two pipelines:

1. **Pipeline 1:** Backfill derived tables from `RawMetrics` (Phase 2)
2. **Pipeline 2:** Backfill enrichment columns from OneBranch `Logs` + `TimelineRecords` into `RawMetrics` (Phase 4)

---

## Table of Contents

1. [Cluster and Database References](#cluster-and-database-references)
2. [Linked Services](#linked-services)
3. [Pipeline 1: Backfill Derived Tables from RawMetrics](#pipeline-1-backfill-derived-tables-from-rawmetrics)
4. [Pipeline 2: Backfill Enrichment Columns from Logs](#pipeline-2-backfill-enrichment-columns-from-logs)
5. [Stored Functions](#stored-functions)
6. [Operational Runbook](#operational-runbook)

---

## Cluster and Database References

| Alias | Cluster URL | Database |
|-------|-------------|----------|
| GatingTest | `https://m365gating-kusto-test.centralus.kusto.windows.net` | `M365GatingAnalytics` |
| GatingPPE | `https://m365gating-kusto-ppe.centralus.kusto.windows.net` | `M365GatingAnalytics` |
| GatingProd | `https://m365gating-kusto-prod.centralus.kusto.windows.net` | `M365GatingAnalytics` |
| OneBranch | `https://onebranchm365release.eastus.kusto.windows.net` | `onebranchreleasetelemetry` |

---

## Linked Services

Two ADF Linked Services are required:

### 1. ADX Linked Service — Gating Cluster

```json
{
    "name": "ls_adx_m365gating",
    "type": "Microsoft.DataFactory/factories/linkedservices",
    "properties": {
        "type": "AzureDataExplorer",
        "typeProperties": {
            "endpoint": "@{pipeline().parameters.GatingClusterUrl}",
            "database": "M365GatingAnalytics",
            "tenant": "<tenant-id>",
            "servicePrincipalId": "<sp-id>",
            "servicePrincipalKey": {
                "type": "AzureKeyVaultSecret",
                "store": { "referenceName": "ls_keyvault", "type": "LinkedServiceReference" },
                "secretName": "adx-sp-key"
            }
        }
    }
}
```

### 2. ADX Linked Service — OneBranch Cluster

```json
{
    "name": "ls_adx_onebranchm365release",
    "type": "Microsoft.DataFactory/factories/linkedservices",
    "properties": {
        "type": "AzureDataExplorer",
        "typeProperties": {
            "endpoint": "https://onebranchm365release.eastus.kusto.windows.net",
            "database": "onebranchreleasetelemetry",
            "tenant": "<tenant-id>",
            "servicePrincipalId": "<sp-id>",
            "servicePrincipalKey": {
                "type": "AzureKeyVaultSecret",
                "store": { "referenceName": "ls_keyvault", "type": "LinkedServiceReference" },
                "secretName": "adx-sp-key"
            }
        }
    }
}
```

---

## Pipeline 1: Backfill PolicyVersionMetrics and RuleExecutionMetrics from RawMetrics

This pipeline only touches the two tables whose schemas changed in Phase 1: **PolicyVersionMetrics** and **RuleExecutionMetrics**. GatingRequestMetrics and PolicyDomainMetrics are left untouched — their schemas are unchanged and their existing data + update policies remain valid.

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| GatingClusterUrl | String | (required) | Target Gating Kusto cluster URL |
| BackfillStartDate | String | (required) | ISO 8601 start date, e.g. `2026-01-29T00:00:00Z` |
| BackfillEndDate | String | (required) | ISO 8601 end date, e.g. `2026-03-30T00:00:00Z` |
| ChunkSizeHours | Int | 24 | Time window per chunk (hours). Use 1 for PolicyVersionMetrics |

### Pipeline Definition

> **Note:** The copy-pasteable pipeline JSON is in [`pl_backfill_gating_normalized_tables.json`](pl_backfill_gating_normalized_tables.json). The JSON below is the reference design.

```json
{
    "name": "pl_backfill_gating_normalized_tables",
    "properties": {
        "parameters": {
            "GatingClusterUrl": { "type": "String" },
            "BackfillStartDate": { "type": "String" },
            "BackfillEndDate": { "type": "String" },
            "ChunkSizeHours": { "type": "Int", "defaultValue": 24 }
        },
        "activities": [
            // --- Disable update policies (PVM and REM only) ---
            // One activity per .alter command (ADX only supports one mgmt command per request)
            { "name": "DisableUpdatePolicy_PolicyVersionMetrics", "type": "AzureDataExplorerCommand", "..." : "..." },
            { "name": "DisableUpdatePolicy_RuleExecutionMetrics", "type": "AzureDataExplorerCommand", "..." : "..." },

            // --- Drop and recreate with new schemas ---
            { "name": "DropTable_PolicyVersionMetrics", "type": "AzureDataExplorerCommand", "..." : "..." },
            { "name": "DropTable_RuleExecutionMetrics", "type": "AzureDataExplorerCommand", "..." : "..." },
            { "name": "CreateTable_PolicyVersionMetrics", "..." : "new schema: + PolicyDomain, BuildId, PolicyRan, PolicyPassed, Data, MetaData" },
            { "name": "CreateTable_RuleExecutionMetrics", "..." : "new schema: + PolicyDomain, PolicyName, PolicyCompliantStatus" },

            // --- Generate time chunks and backfill sequentially (PVM first, then REM) ---
            { "name": "GenerateChunks", "type": "SetVariable", "..." : "..." },
            { "name": "ForEachChunk_PolicyVersionMetrics", "type": "ForEach", "batchCount": 2, "retry": 4, "..." : "..." },
            { "name": "ForEachChunk_RuleExecutionMetrics", "type": "ForEach", "batchCount": 3, "retry": 4, "dependsOn": "ForEachChunk_PolicyVersionMetrics" },

            // --- Re-enable update policies with new queries ---
            { "name": "ReEnableUpdatePolicy_PolicyVersionMetrics", "type": "AzureDataExplorerCommand", "..." : "..." },
            { "name": "ReEnableUpdatePolicy_RuleExecutionMetrics", "type": "AzureDataExplorerCommand", "..." : "..." },

            // --- Validation ---
            { "name": "Validate_RowCounts", "type": "Lookup", "..." : "check PVM and REM row counts + PolicyDomain population" },
            { "name": "CheckValidation_TablesHaveRows", "type": "IfCondition", "..." : "fail if PVM or REM has 0 rows" },
            { "name": "CheckValidation_PolicyDomainPopulated", "type": "IfCondition", "..." : "fail if empty PolicyDomain found" },
            { "name": "Validate_UpdatePoliciesEnabled", "type": "Lookup", "..." : "check PVM and REM update policies are enabled" },
            { "name": "CheckValidation_UpdatePoliciesEnabled", "type": "IfCondition", "..." : "fail if any policy still disabled" }
        ],
        "variables": {
            "ChunkArray": { "type": "Array" }
        }
    }
}
```

### Pipeline Flow

```
DisableUpdatePolicy_PolicyVersionMetrics
        |
DisableUpdatePolicy_RuleExecutionMetrics
        |
DropTable_PolicyVersionMetrics
        |
DropTable_RuleExecutionMetrics
        |
CreateTable_PolicyVersionMetrics
        |
CreateTable_RuleExecutionMetrics
        |
ReEnableUpdatePolicy_PolicyVersionMetrics      ← re-enabled BEFORE backfill
        |
ReEnableUpdatePolicy_RuleExecutionMetrics      ← live ingestion starts populating new tables
        |
  GenerateChunks
        |
ForEachChunk_PolicyVersionMetrics    (batchCount: 2, retry: 4 @ 60s)
        |
ForEachChunk_RuleExecutionMetrics   (batchCount: 3, retry: 4 @ 60s)
        |
  Validate_RowCounts (Lookup)
        |
  CheckValidation_TablesHaveRows
        |
  CheckValidation_PolicyDomainPopulated
        |
  Validate_UpdatePoliciesEnabled (Lookup)
        |
  CheckValidation_UpdatePoliciesEnabled
```

> **Note:** GatingRequestMetrics and PolicyDomainMetrics are **not touched** — their schemas are unchanged.
> Update policies are re-enabled immediately after table creation, before the backfill starts.
> This means live ingestion populates PVM/REM with the new schema while the backfill runs.
> The backfill uses `.set-or-append` (writes directly to PVM/REM, not through RawMetrics),
> so it does NOT trigger update policies — no double-counting of backfill data.
> Set `BackfillEndDate` to a few minutes before pipeline start to avoid duplicate rows
> at the boundary where backfill and live update policy overlap.
> Tables are backfilled sequentially (PVM then REM) to avoid Kusto throttling.
> Each chunk command retries up to 4 times with 60s intervals.

### Recommended Chunk Sizes

| Tables | ChunkSizeHours | batchCount | Reason |
|--------|---------------|------------|--------|
| PolicyVersionMetrics | 1 | 2 | ~25x expansion ratio, needs 1-hour chunks with low parallelism |
| RuleExecutionMetrics | 24 | 3 | Lower expansion ratio, 1-day chunks are fine |

Alternatively, split into separate pipelines per table for independent control.

---

## Pipeline 2: Backfill Enrichment Columns from Logs

This pipeline extracts enrichment data (Ring, DeploymentType, Trainset, Namespace, IsCosmicService, Ev2ServiceType) from the OneBranch cluster and updates `RawMetrics` in the Gating cluster.

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| GatingClusterUrl | String | (required) | Target Gating Kusto cluster URL |
| BackfillStartDate | String | (required) | ISO 8601 start date |
| BackfillEndDate | String | (required) | ISO 8601 end date |
| ChunkSizeDays | Int | 1 | Days per chunk |

### Strategy

Since we cannot directly update individual columns in an existing Kusto table, the approach is:

1. **Extract** enrichment data from OneBranch into a staging table in the Gating cluster
2. **Rebuild** `RawMetrics` by joining existing data with the staging enrichment
3. **Re-run** Pipeline 1 to propagate enriched data into derived tables

### Pipeline Definition

```json
{
    "name": "pl_backfill_enrichment_from_logs",
    "properties": {
        "parameters": {
            "GatingClusterUrl": { "type": "String" },
            "BackfillStartDate": { "type": "String" },
            "BackfillEndDate": { "type": "String" },
            "ChunkSizeDays": { "type": "Int", "defaultValue": 1 }
        },
        "activities": [
            {
                "name": "CreateStagingTable",
                "type": "AzureDataExplorerCommand",
                "linkedServiceName": { "referenceName": "ls_adx_m365gating", "type": "LinkedServiceReference" },
                "typeProperties": {
                    "command": ".drop table EnrichmentStaging ifexists\n.create table EnrichmentStaging (CorrelationId: string, Ring: string, DeploymentType: string, Trainset: string, Namespace: string, IsCosmicService: bool, Ev2ServiceType: string)"
                }
            },
            {
                "name": "GenerateChunks",
                "type": "SetVariable",
                "dependsOn": [{ "activity": "CreateStagingTable", "dependencyConditions": ["Succeeded"] }],
                "typeProperties": {
                    "variableName": "ChunkArray",
                    "value": {
                        "type": "Expression",
                        "value": "@range(0, div(sub(ticks(pipeline().parameters.BackfillEndDate), ticks(pipeline().parameters.BackfillStartDate)), mul(pipeline().parameters.ChunkSizeDays, 864000000000)))"
                    }
                }
            },
            {
                "name": "ForEachChunk_ExtractFromLogs",
                "type": "ForEach",
                "dependsOn": [{ "activity": "GenerateChunks", "dependencyConditions": ["Succeeded"] }],
                "typeProperties": {
                    "isSequential": false,
                    "batchCount": 3,
                    "items": { "value": "@variables('ChunkArray')", "type": "Expression" },
                    "activities": [
                        {
                            "name": "Extract_MOBR_And_Cosmic_Enrichment",
                            "description": "Extract ring, deploymentType, trainset, namespace from Logs; Ev2ServiceType from TimelineRecords. Ingest into staging table in Gating cluster.",
                            "type": "Copy",
                            "typeProperties": {
                                "source": {
                                    "type": "AzureDataExplorerSource",
                                    "query": {
                                        "type": "Expression",
                                        "value": "@concat('LogsEnrichment_Extract(datetime(', addDays(pipeline().parameters.BackfillStartDate, item()), '), datetime(', addDays(pipeline().parameters.BackfillStartDate, add(item(), pipeline().parameters.ChunkSizeDays)), '))')"
                                    },
                                    "queryTimeout": "00:30:00"
                                },
                                "sink": {
                                    "type": "AzureDataExplorerSink",
                                    "ingestionMappingName": "EnrichmentStagingMapping",
                                    "flushImmediately": true
                                }
                            },
                            "inputs": [{
                                "referenceName": "ds_adx_onebranchm365release",
                                "type": "DatasetReference"
                            }],
                            "outputs": [{
                                "referenceName": "ds_adx_m365gating_staging",
                                "type": "DatasetReference"
                            }]
                        }
                    ]
                }
            },
            {
                "name": "RebuildRawMetricsWithEnrichment",
                "description": "Join RawMetrics with EnrichmentStaging and replace RawMetrics content. Chunked to avoid memory issues.",
                "type": "AzureDataExplorerCommand",
                "dependsOn": [{ "activity": "ForEachChunk_ExtractFromLogs", "dependencyConditions": ["Succeeded"] }],
                "linkedServiceName": { "referenceName": "ls_adx_m365gating", "type": "LinkedServiceReference" },
                "typeProperties": {
                    "command": {
                        "type": "Expression",
                        "value": "@concat('.set-or-replace RawMetrics <| RawMetrics_EnrichFromStaging(datetime(', pipeline().parameters.BackfillStartDate, '), datetime(', pipeline().parameters.BackfillEndDate, '))')"
                    }
                }
            },
            {
                "name": "DropStagingTable",
                "type": "AzureDataExplorerCommand",
                "dependsOn": [{ "activity": "RebuildRawMetricsWithEnrichment", "dependencyConditions": ["Succeeded"] }],
                "linkedServiceName": { "referenceName": "ls_adx_m365gating", "type": "LinkedServiceReference" },
                "typeProperties": {
                    "command": ".drop table EnrichmentStaging ifexists"
                }
            },
            {
                "name": "ReRunDerivedTableBackfill",
                "description": "Re-run Pipeline 1 to propagate enrichment columns into derived tables",
                "type": "ExecutePipeline",
                "dependsOn": [{ "activity": "DropStagingTable", "dependencyConditions": ["Succeeded"] }],
                "typeProperties": {
                    "pipeline": { "referenceName": "pl_backfill_gating_normalized_tables", "type": "PipelineReference" },
                    "parameters": {
                        "GatingClusterUrl": { "value": "@pipeline().parameters.GatingClusterUrl", "type": "Expression" },
                        "BackfillStartDate": { "value": "@pipeline().parameters.BackfillStartDate", "type": "Expression" },
                        "BackfillEndDate": { "value": "@pipeline().parameters.BackfillEndDate", "type": "Expression" }
                    },
                    "waitOnCompletion": true
                }
            }
        ],
        "variables": {
            "ChunkArray": { "type": "Array" }
        }
    }
}
```

### Pipeline Flow

```
CreateStagingTable
        |
  GenerateChunks
        |
ForEachChunk_ExtractFromLogs       (parallel: extract from OneBranch → staging table)
        |
RebuildRawMetricsWithEnrichment    (join staging with RawMetrics, replace)
        |
DropStagingTable
        |
ReRunDerivedTableBackfill          (calls Pipeline 1 to repopulate normalized tables)
```

---

## Stored Functions

These KQL functions must be created in the `M365GatingAnalytics` database before running the pipelines. They encapsulate the backfill logic and are called by the ADF pipeline via `.set-or-append <table> <| FunctionName(start, end)`.

### Create all stored functions

Run this as a management command against the Gating cluster:

```kql
// ============================================================
// Function 1: GatingRequestMetrics_Backfill
// ============================================================
.create-or-alter function
    with (folder="Backfill", docstring="Backfill GatingRequestMetrics from RawMetrics for a time window")
    GatingRequestMetrics_Backfill(startTime: datetime, endTime: datetime)
{
    RawMetrics
    | where MetricCreationTime >= startTime and MetricCreationTime < endTime
    | project
        CorrelationId,
        InsertedAt = MetricCreationTime,
        MetricCreationTime,
        OrganizationName,
        ProjectName,
        ProjectId,
        DefinitionId,
        BuildId,
        ServiceTreeId,
        StageName,
        StageAttempt,
        JobAttempt,
        Cloud,
        IsProduction,
        GateType,
        OverallGatingCompliantStatus,
        Metadata
}

// ============================================================
// Function 2: PolicyDomainMetrics_Backfill
// ============================================================
.create-or-alter function
    with (folder="Backfill", docstring="Backfill PolicyDomainMetrics from RawMetrics for a time window")
    PolicyDomainMetrics_Backfill(startTime: datetime, endTime: datetime)
{
    RawMetrics
    | where MetricCreationTime >= startTime and MetricCreationTime < endTime
    | mv-expand DomainResult = RawGatingResult.DomainResults
    | where isnotnull(DomainResult)
    | project
        CorrelationId,
        InsertedAt = MetricCreationTime,
        PolicyDomain = tostring(DomainResult.PolicyDomain),
        DomainCompliantStatus = tostring(DomainResult.DomainCompliantStatus)
}

// ============================================================
// Function 3: PolicyVersionMetrics_Backfill
// NEW SCHEMA: includes PolicyDomain, BuildId, PolicyRan, PolicyPassed, Data, MetaData
// ============================================================
.create-or-alter function
    with (folder="Backfill", docstring="Backfill PolicyVersionMetrics (new schema) from RawMetrics for a time window")
    PolicyVersionMetrics_Backfill(startTime: datetime, endTime: datetime)
{
    let sdpPolicies = dynamic(["ring-bake-time", "stage-bake-time", "ring-progression", "min-stage-count"]);
    // Branch 1: V2 structure (DomainResults → PolicyResults → PolicyVersionResults → RuleResults)
    RawMetrics
    | where MetricCreationTime >= startTime and MetricCreationTime < endTime
    | mv-expand DomainResult = RawGatingResult.DomainResults
    | where isnotnull(DomainResult)
    | mv-expand PolicyResult = DomainResult.PolicyResults
    | where isnotnull(PolicyResult)
    | mv-expand PolicyVersionResult = PolicyResult.PolicyVersionResults
    | where isnotnull(PolicyVersionResult)
    | mv-expand RuleResult = PolicyVersionResult.RuleResults
    | where isnotnull(RuleResult)
    | project
        CorrelationId,
        InsertedAt = MetricCreationTime,
        PolicyDomain = tostring(DomainResult.PolicyDomain),
        PolicyName = tostring(PolicyResult.PolicyName),
        Version = tostring(PolicyVersionResult.Version),
        PolicyMode = tostring(PolicyVersionResult.PolicyMode),
        BuildId = tostring(RuleResult.BuildId),
        PolicyRan = tobool(RuleResult.PolicyRan),
        PolicyPassed = tobool(RuleResult.PolicyPassed),
        Data = tostring(RuleResult.Data),
        MetaData = RuleResult.MetaData
    | union (
        // Branch 2: V1 flat structure (PolicyResults → RuleResults, no version nesting) — derive PolicyDomain from PolicyName
        RawMetrics
        | where MetricCreationTime >= startTime and MetricCreationTime < endTime
        | mv-expand PolicyResult = RawGatingResult.PolicyResults
        | where isnotnull(PolicyResult)
        | mv-expand RuleResult = PolicyResult.RuleResults
        | where isnotnull(RuleResult)
        | extend _policyName = tostring(PolicyResult.PolicyName)
        | project
            CorrelationId,
            InsertedAt = MetricCreationTime,
            PolicyDomain = iff(_policyName in (sdpPolicies), "SafeDeploymentPolicies", "ChangeManagementPolicies"),
            PolicyName = _policyName,
            Version = "default",
            PolicyMode = tostring(PolicyResult.PolicyMode),
            BuildId = tostring(RuleResult.BuildId),
            PolicyRan = tobool(RuleResult.PolicyRan),
            PolicyPassed = tobool(RuleResult.PolicyPassed),
            Data = tostring(RuleResult.Data),
            MetaData = RuleResult.MetaData
    )
}

// ============================================================
// Function 4: RuleExecutionMetrics_Backfill
// NEW SCHEMA: policy-level with PolicyDomain, PolicyName, PolicyCompliantStatus
// ============================================================
.create-or-alter function
    with (folder="Backfill", docstring="Backfill RuleExecutionMetrics (new schema) from RawMetrics for a time window")
    RuleExecutionMetrics_Backfill(startTime: datetime, endTime: datetime)
{
    let sdpPolicies = dynamic(["ring-bake-time", "stage-bake-time", "ring-progression", "min-stage-count"]);
    // Branch 1: V2 structure (DomainResults → PolicyResults) — PolicyDomain is explicit
    RawMetrics
    | where MetricCreationTime >= startTime and MetricCreationTime < endTime
    | mv-expand DomainResult = RawGatingResult.DomainResults
    | where isnotnull(DomainResult)
    | mv-expand PolicyResult = DomainResult.PolicyResults
    | where isnotnull(PolicyResult)
    | project
        CorrelationId,
        InsertedAt = MetricCreationTime,
        BuildId,
        PolicyDomain = tostring(DomainResult.PolicyDomain),
        PolicyName = tostring(PolicyResult.PolicyName),
        PolicyCompliantStatus = tostring(PolicyResult.PolicyCompliantStatus)
    | union (
        // Branch 2: V1 flat structure (PolicyResults, no DomainResults wrapper) — derive PolicyDomain from PolicyName
        RawMetrics
        | where MetricCreationTime >= startTime and MetricCreationTime < endTime
        | mv-expand PolicyResult = RawGatingResult.PolicyResults
        | where isnotnull(PolicyResult)
        | extend _policyName = tostring(PolicyResult.PolicyName)
        | project
            CorrelationId,
            InsertedAt = MetricCreationTime,
            BuildId,
            PolicyDomain = iff(_policyName in (sdpPolicies), "SafeDeploymentPolicies", "ChangeManagementPolicies"),
            PolicyName = _policyName,
            PolicyCompliantStatus = tostring(PolicyResult.PolicyCompliantStatus)
    )
}

// ============================================================
// Function 5: LogsEnrichment_Extract
// Extracts Ring, DeploymentType, Trainset, Namespace, IsCosmicService
// from OneBranch Logs + Ev2ServiceType from TimelineRecords
// Must be created on the OneBranch cluster (onebranchreleasetelemetry)
// ============================================================
// NOTE: This function is created on the OneBranch cluster, not the Gating cluster.
//       See the "OneBranch Stored Functions" section below.

// ============================================================
// Function 6: RawMetrics_EnrichFromStaging
// Joins RawMetrics with EnrichmentStaging to populate enrichment columns
// ============================================================
.create-or-alter function
    with (folder="Backfill", docstring="Join RawMetrics with EnrichmentStaging to populate enrichment columns")
    RawMetrics_EnrichFromStaging(startTime: datetime, endTime: datetime)
{
    RawMetrics
    | where MetricCreationTime >= startTime and MetricCreationTime < endTime
    | join kind=leftouter (
        EnrichmentStaging
    ) on CorrelationId
    | project
        MetricCreationTime,
        CorrelationId,
        OrganizationName,
        ProjectName,
        ProjectId,
        DefinitionId,
        BuildId,
        ServiceTreeId,
        StageName,
        StageAttempt,
        JobAttempt,
        Cloud,
        IsProduction,
        GateType,
        OverallGatingCompliantStatus,
        Metadata,
        RawGatingResult,
        // Enrichment columns: prefer staging values, fall back to existing
        Trainset = coalesce(EnrichmentStaging_Trainset, Trainset),
        DeploymentType = coalesce(EnrichmentStaging_DeploymentType, DeploymentType),
        Ring = coalesce(EnrichmentStaging_Ring, Ring),
        Namespace = coalesce(EnrichmentStaging_Namespace, Namespace),
        IsCosmicService = coalesce(EnrichmentStaging_IsCosmicService, IsCosmicService),
        Ev2ServiceType = coalesce(EnrichmentStaging_Ev2ServiceType, Ev2ServiceType)
    | project-away EnrichmentStaging_*
}
```

### OneBranch Stored Functions

This function must be created on the **OneBranch cluster** (`onebranchm365release.eastus.kusto.windows.net` / `onebranchreleasetelemetry`):

```kql
// ============================================================
// Function 5: LogsEnrichment_Extract
// Created on: onebranchm365release / onebranchreleasetelemetry
// Extracts all enrichment columns for a time window
// Returns one row per CorrelationId-equivalent key
// ============================================================
.create-or-alter function
    with (folder="Backfill", docstring="Extract enrichment data from Logs and TimelineRecords for gating backfill")
    LogsEnrichment_Extract(startTime: datetime, endTime: datetime)
{
    // --- Step 1: Extract MOBR API params and Cosmic data from Logs ---
    let mobrAndCosmic =
        Logs
        | where todatetime(Timestamp) >= startTime and todatetime(Timestamp) < endTime
        | where TaskId == "d98bb041-d191-41c2-b770-3dc3e7b10d7e"
        | where Message has "/api/GetMOBRDeploymentCompliantStatusByBuildId"
            or Message has "NamespaceRingJson:"
        | extend AdoAccount = tolower(AdoAccount)
        | extend MsgType = iff(Message has "/api/GetMOBRDeploymentCompliantStatusByBuildId", "MOBR", "Cosmic")
        // MOBR params extraction
        | extend _ring = iff(MsgType == "MOBR",
            tostring(extract(@"[?&]ring=([^&\s]+)", 1, Message)), "")
        | extend Ring = iif(isempty(_ring) or tolower(_ring) == "null", "", toupper(_ring))
        | extend _depType = iff(MsgType == "MOBR",
            tostring(extract(@"[?&]deploymentType=([^&\s]+)", 1, Message)), "")
        | extend DeploymentType = iif(isempty(_depType) or tolower(_depType) == "null", "", _depType)
        | extend _trainset = iff(MsgType == "MOBR",
            tostring(extract(@"[?&]trainset=([^&\s]+)", 1, Message)), "")
        | extend Trainset = iif(isempty(_trainset) or tolower(_trainset) == "null", "", _trainset)
        // Cosmic params extraction
        | extend CosmicRing = iff(MsgType == "Cosmic",
            toupper(extract(@"(?i)\\""ring\\""\s*:\s*\\""([^\\""]*)\\""", 1, Message)), "")
        | extend CosmicNamespace = iff(MsgType == "Cosmic",
            extract(@"(?i)\\""namespace\\""\s*:\s*\\""([^\\""]*)\\""", 1, Message), "")
        // Aggregate per stage run
        | summarize
            Ring = take_anyif(Ring, Ring != ""),
            DeploymentType = take_anyif(DeploymentType, DeploymentType != ""),
            Trainset = take_anyif(Trainset, Trainset != ""),
            CosmicRing = take_anyif(CosmicRing, CosmicRing != ""),
            CosmicNamespace = take_anyif(CosmicNamespace, CosmicNamespace != "")
            by AdoAccount, ProjectId, Id, Environment
        // Finalize ring and namespace
        | extend Ring = coalesce(iff(isempty(Ring), CosmicRing, Ring), "")
        | extend Ring = iff(Ring contains "$" or Ring contains ",", "", trim(" ", Ring))
        | extend Namespace = iff(isempty(CosmicNamespace), "Non-Cosmic", CosmicNamespace)
        | extend IsCosmicService = isnotempty(CosmicRing);
    // --- Step 2: Extract Ev2ServiceType from TimelineRecords ---
    let ev2Types =
        TimelineRecords
        | where todatetime(StartTime) >= startTime and todatetime(StartTime) < endTime
        | summarize
            HasClassic = max(iff(tostring(parse_json(Task).Name) == "ExpressV2Internal", 1, 0)),
            HasRA = max(iff(tostring(parse_json(Task).Name) == "Ev2RARollout", 1, 0))
            by AdoAccount, ProjectId, Id, Environment
        | extend Ev2ServiceType = case(
            HasClassic == 1 and HasRA == 0, "RS Only",
            HasClassic == 0 and HasRA == 1, "RA Only",
            HasClassic == 1 and HasRA == 1, "Hybrid Services",
            "Unknown")
        | project AdoAccount = tolower(AdoAccount), ProjectId, Id, Environment, Ev2ServiceType;
    // --- Step 3: Join and build CorrelationId to match RawMetrics ---
    mobrAndCosmic
    | join kind=leftouter ev2Types on AdoAccount, ProjectId, Id, Environment
    // Build CorrelationId matching RawMetrics format: {org}_{projectId}_{buildId}_{stageName}_{attempt}
    // We extract BuildId from PipelineRecords to get the full key
    | join kind=leftouter (
        PipelineRecords
        | where todatetime(Timestamp) >= startTime and todatetime(Timestamp) < endTime
        | project AdoAccount = tolower(AdoAccount), ProjectId, Id, BuildId = Id, DefinitionId
    ) on AdoAccount, ProjectId, Id
    | extend CorrelationId = strcat(AdoAccount, "_", ProjectId, "_", Id, "_", Environment, "_1")
    | project
        CorrelationId,
        Ring,
        DeploymentType,
        Trainset,
        Namespace,
        IsCosmicService,
        Ev2ServiceType = coalesce(Ev2ServiceType, "Unknown")
}
```

> **Important:** The `CorrelationId` construction in `LogsEnrichment_Extract` must match the format used in `RawMetrics`. The pattern `{org}_{projectId}_{buildId}_{stageName}_{attempt}` was observed in prod data. Verify this matches by comparing a sample before running at scale.

---

## Operational Runbook

### Pre-Flight Checks

Before running either pipeline:

1. **Verify cluster access:**
   ```bash
   az account show
   # Ensure logged-in identity has Kusto admin (or at minimum ingestor + viewer) on target clusters
   ```

2. **Verify RawMetrics data range:**
   ```kql
   RawMetrics | summarize min(MetricCreationTime), max(MetricCreationTime), count()
   ```

3. **Verify CorrelationId format matches Logs join key:**
   ```kql
   RawMetrics | take 10 | project CorrelationId
   // Compare with: LogsEnrichment_Extract(ago(1d), now()) | take 10 | project CorrelationId
   ```

4. **Create stored functions** (see above) on both clusters before first run.

### Execution Order

```
Step 1: Create stored functions on Gating cluster (Functions 1-4, 6)
Step 2: Create stored function on OneBranch cluster (Function 5)
Step 3: Run Pipeline 1 (backfill derived tables from RawMetrics)
        Parameters:
          GatingClusterUrl = <target cluster>
          BackfillStartDate = "2026-01-29T00:00:00Z"
          BackfillEndDate = "2026-03-30T00:00:00Z"
          ChunkSizeHours = 24
        Run again with ChunkSizeHours = 1 for PolicyVersionMetrics if needed
Step 4: Verify derived table row counts
Step 5: Deploy code changes (Phase 3 — new enrichment columns in GatingWorkflowBase)
Step 6: Deploy Bicep changes (Phase 3 — RawMetrics schema + mapping)
Step 7: Run Pipeline 2 (backfill enrichment from Logs)
        Parameters:
          GatingClusterUrl = <target cluster>
          BackfillStartDate = "2026-01-29T00:00:00Z"
          BackfillEndDate = "2026-03-30T00:00:00Z"
          ChunkSizeDays = 1
Step 8: Verify enrichment columns populated in RawMetrics
Step 9: Verify derived tables have enrichment columns (re-backfilled by Pipeline 2)
```

### Validation Queries

Run after Pipeline 1 completes (also automated as pipeline validation activities):

```kql
// Row counts for backfilled tables
union
    (PolicyVersionMetrics | count | extend Table="PolicyVersionMetrics"),
    (RuleExecutionMetrics | count | extend Table="RuleExecutionMetrics")
| project Table, Count

// Verify PolicyVersionMetrics new columns populated
PolicyVersionMetrics
| where isnotempty(BuildId)
| take 5
| project CorrelationId, PolicyDomain, PolicyName, Version, PolicyMode, BuildId, PolicyRan, PolicyPassed

// Verify RuleExecutionMetrics new schema
RuleExecutionMetrics
| where isnotempty(PolicyName)
| take 5
| project CorrelationId, BuildId, PolicyDomain, PolicyName, PolicyCompliantStatus

// Verify PolicyDomain is populated (should be 0 empty rows)
PolicyVersionMetrics | where isempty(PolicyDomain) | count
RuleExecutionMetrics | where isempty(PolicyDomain) | count

// Verify update policies are re-enabled
.show table * policy update
| where EntityName has "PolicyVersionMetrics" or EntityName has "RuleExecutionMetrics"
| extend Policy = parse_json(tostring(parse_json(Policy)[0]))
| project EntityName, IsEnabled = tobool(Policy.IsEnabled)

// Verify enrichment columns in RawMetrics (after Pipeline 2)
RawMetrics
| where isnotempty(Ring) or isnotempty(Trainset)
| take 5
| project CorrelationId, Ring, DeploymentType, Trainset, Namespace, IsCosmicService, Ev2ServiceType

// Enrichment coverage
RawMetrics
| where MetricCreationTime > ago(60d)
| summarize
    Total = count(),
    HasRing = countif(isnotempty(Ring)),
    HasTrainset = countif(isnotempty(Trainset)),
    HasDeploymentType = countif(isnotempty(DeploymentType)),
    HasEv2ServiceType = countif(Ev2ServiceType != "Unknown" and isnotempty(Ev2ServiceType))
| extend
    RingCoverage = round(100.0 * HasRing / Total, 1),
    TrainsetCoverage = round(100.0 * HasTrainset / Total, 1),
    DeploymentTypeCoverage = round(100.0 * HasDeploymentType / Total, 1),
    Ev2ServiceTypeCoverage = round(100.0 * HasEv2ServiceType / Total, 1)
```

### Failure Recovery

| Failure Point | Recovery |
|---------------|----------|
| Pipeline 1 fails mid-backfill | Safe to re-run — `.set-or-append` is additive. Drop and recreate PVM/REM first to avoid duplicates, then re-run |
| Pipeline 2 fails during Logs extraction | Safe to re-run — staging table is dropped and recreated at start |
| Pipeline 2 fails during `.set-or-replace` | Re-run entire pipeline — staging data will be re-extracted |
| CorrelationId mismatch | Fix `LogsEnrichment_Extract` join key construction, drop staging table, re-run Pipeline 2 |
| Kusto memory errors on large chunks | Reduce `ChunkSizeHours` (Pipeline 1) or `ChunkSizeDays` (Pipeline 2) and re-run |



What ADF even is

  Azure Data Factory is basically a drag-and-drop orchestration tool in the Azure portal. You create "pipelines" which are sequences of activities — think of it
  like a workflow that runs Kusto commands, copies data between systems, loops over things, etc. You don't write code — you configure JSON definitions through a
  UI (or commit JSON to a repo).

  Where to start

  1. Get access

  You need:
  - Access to the Azure portal and the ADF instance (ask your boss which resource group / ADF factory name)
  - Kusto admin or ingestor permissions on the M365 Gating clusters (test, PPE, prod)
  - Kusto viewer permissions on the OneBranch cluster (onebranchm365release)
  - A service principal (SP) or managed identity that ADF will use to authenticate to both Kusto clusters — this likely already exists if there are other ADF
  pipelines

  2. Set up Linked Services

  In the ADF portal (Author tab → Manage → Linked Services), create two connections:
  - One pointing to the Gating Kusto cluster
  - One pointing to the OneBranch Kusto cluster

  These are like "connection strings" — they tell ADF how to talk to each cluster. You pick "Azure Data Explorer" as the type, paste the cluster URL, and
  configure auth (usually a service principal from Key Vault).

  3. Create the stored functions FIRST

  Before building any pipeline, you need to manually create the KQL functions on the Kusto clusters. Open the Kusto Web Explorer (or Azure Data Explorer portal),
   connect to each cluster, and run the .create-or-alter function commands from the document. These are just saved KQL queries that the pipeline will call by
  name.

  - Functions 1-4 and 6 → run on the Gating cluster
  - Function 5 → run on the OneBranch cluster

  You can test each function manually first: GatingRequestMetrics_Backfill(ago(1h), now()) | count — make sure it returns data and doesn't error.

  4. Build Pipeline 1 in the ADF UI

  In the Author tab → Pipelines → New Pipeline:

  1. Add parameters — click on the pipeline canvas background, go to Parameters tab, add GatingClusterUrl, BackfillStartDate, BackfillEndDate, ChunkSizeHours
  2. Add activities by dragging them from the left panel:
    - "Azure Data Explorer Command" activity for DisableUpdatePolicies — paste the .alter table ... policy update commands
    - Another ADX Command for DropAndRecreateTables — paste the .drop table and .create table commands
    - A "Set Variable" activity to generate the chunk array (this creates a list like [0, 1, 2, ... 59] representing each time chunk)
    - A "ForEach" activity for each table — inside each ForEach, put an ADX Command that calls the stored function with the chunk's time window
    - A final ADX Command for ReEnableUpdatePolicies
  3. Wire the dependencies — drag the green arrows between activities to set the execution order (the document shows the flow diagram)
  4. The JSON in the document is what ADF generates behind the scenes. You can either build it in the UI (easier to learn) or switch to the "Code" view and paste
   the JSON directly.

  5. Test on the Test cluster first

  Never run against prod first. Set GatingClusterUrl to the test cluster URL, use a small date range (like 1 day), and validate:
  - Tables get dropped and recreated
  - Chunks execute and data appears
  - Row counts look reasonable
  - Update policies get re-enabled

  6. Build Pipeline 2 similarly

  Same process but it's a cross-cluster pipeline — it reads from OneBranch and writes to Gating. The key difference is the Copy activity (not just ADX Command)
  because you're moving data between two different clusters. In ADF, you configure a Source (OneBranch dataset) and Sink (Gating staging table dataset).

  7. The tricky part: CorrelationId matching

  The hardest part of Pipeline 2 isn't the ADF setup — it's making sure the join key between Logs data and RawMetrics actually matches. The CorrelationId in
  RawMetrics follows a specific format. You need to sample both sides and verify they align before running at scale. If they don't match, the enrichment join
  returns nothing and all your columns stay null.

  What you don't need to worry about

  - You don't need to write any C# code for the backfill
  - You don't need to deploy anything via EV2 — ADF pipelines are managed in the Azure portal
  - The stored functions handle all the complex KQL — the pipeline just calls them with time parameters

  Common gotchas

  - Kusto memory limits — if a chunk is too large, the .set-or-append will fail with an OOM error. Reduce chunk size and retry
  - Timeouts — set the activity timeout in ADF to at least 30 minutes for large chunks
  - Parallelism — the batchCount in ForEach controls how many chunks run simultaneously. Start low (3) and increase if the cluster handles it
  - Don't forget to re-enable update policies — if the pipeline fails midway and you don't re-enable them, new ingestions won't populate the derived tables. Live
   data stops flowing.