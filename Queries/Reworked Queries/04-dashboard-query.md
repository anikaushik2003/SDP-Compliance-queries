# SDP Compliance Dashboard Query (Power BI)

This is what Power BI executes. The materialized table is pre-joined and pre-enriched by the daily ADF pipeline — Power BI just reads it with simple filters and joins dimension tables.

No cross-cluster calls. No regex. No JSON parsing. No Logs, TimelineRecords, PipelineRecords, BuildYamlSnapshot.

**Tables:**
- `SDPMaterialized` — fact table (stage-run-policy grain)
- `ServiceClassification` — dimension (service-level, upserted daily)

---

## Query

```kql
// ServiceClassification dimension (upserted daily, ~1.3K rows)
let SvcClass = ServiceClassification
    | project ServiceId, isCosmicService, Ev2ServiceType;
// PipelineOnboarded (computed at query time from fact table)
let PipelineOnboarded =
    SDPMaterialized
    | summarize
        AllStagesOnboarded = min(Onboarded),
        HasTrainset = max(iff(isnotempty(Trainset), 1, 0))
        by PipelineUrl, ServiceId
    | extend PipelineOnboarded = iff(AllStagesOnboarded == 1 and HasTrainset == 1, 1, 0)
    | project PipelineUrl, PipelineOnboarded;
// Main dashboard query
SDPMaterialized
// Service-level classification
| join kind=leftouter SvcClass on ServiceId
// Pipeline-level onboarding
| join kind=leftouter PipelineOnboarded on PipelineUrl
// Final projection — all columns available to Power BI
| project
    // Time
    InsertedAt,
    MetricCreationTime,
    // Identity
    CorrelationId,
    AdoAccount,
    ProjectId,
    DefinitionId,
    BuildId,
    StageName,
    PipelineUrl,
    PipelineKey,
    RunUrl,
    // Policy (row grain)
    PolicyDomain,
    PolicyName,
    PolicyCompliantStatus,
    // Stage-level properties
    Ring,
    Namespace,
    Cloud,
    DeploymentType,
    Trainset,
    isCosmicStage,
    HasLockbox,
    HasClassic,
    HasRA,
    Onboarded,
    // Pipeline-level (computed)
    PipelineOnboarded,
    // Service-level (from dimension table)
    isCosmicService,
    Ev2ServiceType,
    // YAML
    YamlId,
    PipelineName,
    // ServiceTree
    ServiceId,
    Workload,
    DevOwner,
    DivisionName,
    OrganizationName,
    ServiceGroupName,
    TeamGroupName,
    ServiceName
```

---

## Row Count Estimation

Measured from prod cluster on 2026-04-01.

### SDPMaterialized (fact table — SDP only)

| Metric | Value | Source |
|--------|-------|--------|
| Total stage-runs (60d) | ~555K | `RawMetrics \| count()` |
| Release gating requests | ~480K | `GatingRequestMetrics` where `GateType == "Release"` |
| SDP + CM domain coverage | 100% | `PolicyDomainMetrics`: same 477K correlationIds for each domain |
| Stages with SDP V2 policies | ~460K | `PolicyVersionMetrics` SDP `dcount(CorrelationId)` |
| Distinct SDP policy names | 4 | ring-bake-time, ring-progression, stage-bake-time, min-stage-count |
| Avg SDP policies per stage | 2.9 | Range 2–4 (not all services get all 4) |
| **SDPMaterialized rows (60d)** | **~1.4M** | 460K x 2.9 |
| Daily volume | ~23K/day | 1.4M / 60 |
| Distinct builds | ~289K | `dcount(BuildId)` |
| Distinct services | ~1,275 | `dcount(ServiceTreeId)` |
| Distinct ADO orgs | 25 | `dcount(OrganizationName)` |

### CM Policies (not in current scope — for future reference)

| Metric | Value |
|--------|-------|
| Distinct CM policy names | ~25+ |
| Avg CM policies per stage | 15.5 (range 8–78) |
| If added: additional rows (60d) | ~7.4M (480K x 15.5) |
| Total with both domains | ~8.8M |

### ServiceClassification (dimension table)

| Metric | Value |
|--------|-------|
| Rows | ~1,275 (one per ServiceId) |
| Growth | Slow — only grows when new services start using gating |

### Comparison with old approach

| Metric | Old | New |
|--------|-----|-----|
| Grain | stage-run | stage-run-policy |
| Rows (60d) | ~564K | ~1.4M (SDP only) |
| Columns | ~30 | ~30 |
| Cross-cluster joins at dashboard | 10+ | 0 (fact table) + 1 small dimension |
| Regex/JSON parsing | 3 message types | 0 |
| Dashboard query complexity | ~250 lines | Simple read |

### Power BI Feasibility

At ~50 bytes/row avg, 1.4M rows = ~70 MB uncompressed. Well within Power BI Pro (1 GB limit) and DirectQuery limits.
