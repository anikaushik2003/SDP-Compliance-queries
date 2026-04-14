// ============================================================
// SDP Compliance — Daily Slice Query (Parameterized)
// ============================================================
//
// GRAIN: stage-run-policy
//   One row per (CorrelationId, PolicyName).
//   Each stage-run produces N rows — one per SDP policy evaluated.
//
// TIME AXIS: InsertedAt (not MetricCreationTime)
//   - InsertedAt = when Kusto actually ingested the row (set by update policy via now())
//   - MetricCreationTime = when the gating API ran (set in C# before queued ingestion)
//   - These differ: queued ingestion is async, delay can be minutes to hours
//   - All 4 derived tables get the SAME InsertedAt for the same CorrelationId because:
//     (a) All update policies fire on the same RawMetrics ingestion event
//     (b) All are IsTransactional: true — all-or-nothing, no partial ingestion
//     (c) now() is evaluated in the same transaction
//   - This means the "Day 1 vs Day 5" split CANNOT happen:
//     a CorrelationId appears in GatingRequestMetrics and RuleExecutionMetrics
//     in the same InsertedAt window, guaranteed.
//   - Late-arriving data is handled naturally: if gating ran Monday but
//     ingestion was delayed to Wednesday, Wednesday's slice picks up
//     the complete row across all tables. The 2-day sliding window
//     (process D-1 to D+1) covers delays up to ~48 hours.
//
// TIME SLICING:
//   Parameterized by (sliceStart, sliceEnd). Each day's slice is
//   self-contained. Daily results are unioned into the accumulated table:
//     Day 1: process(d1_start, d1_end) → append to table
//     Day 2: process(d2_start, d2_end) → append to table
//     ...
//
// SOURCES:
//   PolicyHub:  m365gating_GatingRequestMetrics (scoping + enrichment)
//               m365gating_RuleExecutionMetrics (policy grain expansion)
//   OneBranch:  BuildYamlMapSnapshot + BuildYamlSnapshot (YAML bridge)
//               ServiceTreeHierarchySnapshot + ServiceTreeSnapshot
//
// SCOPING INVARIANT:
//   PolicyDomain == "SafeDeploymentPolicies" guarantees the stage has
//   lockbox + EV2 + above-ARM. No separate scoping filters needed.
//   Denominator = "stages with evaluated SDP policies", NOT "all
//   in-scope MOBR pipelines".
//
// WHAT'S ELIMINATED:
//   - Logs table (all 3 message types, regex, JSON parsing)
//   - TimelineRecords (task scanning, health checks, below-ARM)
//   - PipelineRecords (scoping, GetVersionInfo)
//   - UniqueStageId (old join key — not needed)
//   - summarize pivot pattern (4 hardcoded status columns)
// ============================================================
// --- Parameters (passed by ADF) ---
// declare query_parameters(sliceStart: datetime, sliceEnd: datetime);
let sliceStart = datetime(2026-03-30);  // replaced by ADF parameter
let sliceEnd   = datetime(2026-03-31);  // replaced by ADF parameter
// --- Exempted Pipelines ---
let exemptedPipelines = dynamic([
    "o365exchange_959adb23-f323-4d52-8203-ff34e5cbeefa_51671",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_36193",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_35003",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_36303",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_36344",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_35235",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_35514",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_35486",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_35638",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_35639",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_35940",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_35941",
    "office_69fbb5e6-b7ff-4fdd-a5e0-228623ef3b0b_34999"
]);
// --- ServiceTree (snapshot — no time filter needed) ---
let sglist = dynamic(["IC3","Yammer","TAOS","O365 FAST","Substrate Platform","Mesh","Microsoft Search Assistants & Intelligence (MSAI)","O365 Enterprise Cloud","Microsoft Teams"]);
let orglist = dynamic(["WebXT","OPG","OneDrive/SharePoint","Data Security and Privacy"]);
let ServiceTree =
    cluster('onebranchm365release.eastus.kusto.windows.net')
        .database('onebranchreleasetelemetry')
        .ServiceTreeHierarchySnapshot
    | where Level == "Service" and ServiceId != ""
    | project DivisionName, OrganizationName, ServiceGroupName, TeamGroupName, ServiceName, ServiceId
    | extend Workload = iif(
        ServiceGroupName in (sglist), ServiceGroupName,
        iif(OrganizationName in (orglist), OrganizationName,
        iif(OrganizationName has "W+D", "W+D", OrganizationName)))
    | distinct ServiceId, Workload, DivisionName, OrganizationName, ServiceGroupName, TeamGroupName, ServiceName
    | join kind=leftouter (
        cluster('onebranchm365release.eastus.kusto.windows.net')
            .database('onebranchreleasetelemetry')
            .ServiceTreeSnapshot
        | distinct ServiceId, DevOwner
    ) on ServiceId
    | distinct ServiceId,
        Workload = iff(Workload == "Skype", "M365 Core -IC3",
                   iff(Workload == "Microsoft Teams", "CAP - Microsoft Teams", Workload)),
        DevOwner, DivisionName, OrganizationName, ServiceGroupName, TeamGroupName, ServiceName;
let ringworkload = datatable(Workload:string, AllowedRings:dynamic)[
    "M365 Core - IC3", dynamic(["NPE","TDF", "SDF", "MSIT", "GENERAL"]),
    "OPG", dynamic(["TEST", "SDF", "MSIT", "PROD", "GCC", "GCCH", "DOD", "GALLATIN", "AG08", "AG09"]),
    "CAP - Microsoft Teams", dynamic(["TEST", "DOGFOOD", "MSFT", "PROD", "GCC", "GCCH", "DOD", "GALLATIN", "AG08", "AG09"]),
    "Substrate Platform", dynamic(["TEST", "SDF", "MSIT", "PROD", "GCC", "GCCH", "DOD", "GALLATIN", "AG08", "AG09"]),
    "Others", dynamic(["TEST", "SDF", "MSIT", "PROD", "GCC", "GCCH", "DOD", "GALLATIN", "AG08", "AG09"])
];
let ringworkloadlist = ringworkload | summarize make_set(Workload);
// =============================================================
// STEP 1: Day's stage-runs from GatingRequestMetrics
//         Time-sliced by InsertedAt (ingestion time)
// =============================================================
let dayRuns =
    m365gating_GatingRequestMetrics
    | where InsertedAt >= sliceStart and InsertedAt < sliceEnd
    | where GateType == "Release"
    | extend AdoAccount = tolower(OrganizationName)
    | extend PipelineUrl = strcat(
        "https://dev.azure.com/", AdoAccount, "/", ProjectId,
        "/_build?definitionId=", DefinitionId, "&_a=summary")
    | extend PipelineKey = strcat(AdoAccount, "_", tolower(tostring(ProjectId)), "_", tostring(DefinitionId))
    | where not(PipelineKey in (exemptedPipelines))
    | project
        CorrelationId, InsertedAt, MetricCreationTime,
        AdoAccount, ProjectId, DefinitionId, BuildId, StageName,
        PipelineUrl, PipelineKey, ServiceTreeId, Cloud,
        // Enrichment columns (direct — no parsing)
        Ring, Namespace, DeploymentType, Trainset,
        // Stage-level flags (NOT service-level — see ServiceClassification table)
        IsCosmicService, Ev2ServiceType;
// =============================================================
// STEP 2: Day's policy results from RuleExecutionMetrics
//         Time-sliced by InsertedAt (same ingestion moment as Step 1
//         — guaranteed by transactional update policies)
//         Filtered to SDP only (scoping invariant)
// =============================================================
let dayPolicies =
    m365gating_RuleExecutionMetrics
    | where InsertedAt >= sliceStart and InsertedAt < sliceEnd
    | where PolicyDomain == "SafeDeploymentPolicies"
    | project CorrelationId, PolicyDomain, PolicyName, PolicyCompliantStatus;
// =============================================================
// STEP 3: Day's YAML mapping (scoped by day's BuildIds)
// =============================================================
let dayYaml =
    dayRuns
    | distinct PipelineUrl, BuildId = tolong(BuildId)
    | join kind=inner (
        cluster('onebranchm365release.eastus.kusto.windows.net')
            .database('onebranchreleasetelemetry')
            .BuildYamlMapSnapshot
        | extend PipelineUrl = strcat(
            "https://dev.azure.com/", tolower(OrganizationName), "/", ProjectId,
            "/_build?definitionId=", DefinitionId, "&_a=summary")
        | project PipelineUrl, BuildId, YamlId
    ) on PipelineUrl, BuildId
    | join kind=leftouter (
        cluster('onebranchm365release.eastus.kusto.windows.net')
            .database('onebranchreleasetelemetry')
            .BuildYamlSnapshot
        | where Index == 0
        | distinct YamlId, PipelineName = DisplayName
    ) on YamlId
    | project PipelineUrl, BuildId, YamlId, PipelineName;
// =============================================================
// ASSEMBLY: dayRuns x dayPolicies x dayYaml x ServiceTree
// =============================================================
dayRuns
// Grain expansion: stage-run → stage-run-policy
| join kind=inner dayPolicies on CorrelationId
// YAML enrichment
| join kind=leftouter dayYaml on PipelineUrl, $left.BuildId == $right.BuildId
// Derive STAGE-LEVEL task flags (not service-level — see ServiceClassification)
| extend HasLockbox = 1  // gating implies lockbox
| extend HasClassic = iff(Ev2ServiceType in ("RS Only", "Hybrid Services"), 1, 0)
| extend HasRA = iff(Ev2ServiceType in ("RA Only", "Hybrid Services"), 1, 0)
| extend isCosmicStage = iff(IsCosmicService == true, 1, 0)
// ServiceTree enrichment
| join kind=leftouter ServiceTree on $left.ServiceTreeId == $right.ServiceId
// Stage-level Onboarded
| extend ringworkload = iff(Workload has_any (ringworkloadlist), Workload, "Others")
| join kind=leftouter ringworkload on $left.ringworkload == $right.Workload
| extend Onboarded = iff(Ring == "", 0, iff(AllowedRings has Ring, 1, 0))
// Construct run URL
| extend RunUrl = strcat(split(PipelineUrl, "?")[0], "/results?buildId=", BuildId, "&view=results")
// Final projection — only columns needed for the materialized view
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
    // Policy (this IS the grain)
    PolicyDomain,
    PolicyName,
    PolicyCompliantStatus,
    // Stage-level enrichment
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
    // YAML
    YamlId,
    PipelineName,
    // ServiceTree
    ServiceId = ServiceTreeId,
    Workload,
    DevOwner,
    DivisionName,
    OrganizationName,
    ServiceGroupName,
    TeamGroupName,
    ServiceName
// =============================================================
// NOTE: Service-level classifications (isCosmicService, Ev2ServiceType,
// PipelineOnboarded) are NOT in this query. They live in a separate
// ServiceClassification table that is upserted daily.
// See 03-service-classification.md
// =============================================================
