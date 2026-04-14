// ============================================================
// ServiceClassification — Dimension Table
// Upserted daily from incoming SDPMaterialized rows
// ============================================================
//
// WHY A SEPARATE TABLE:
//   isCosmicService, Ev2ServiceType, and PipelineOnboarded are
//   service/pipeline-level aggregations that depend on ALL historical
//   data for a ServiceId — not just one day's slice.
//
// WHY UPSERT, NOT RECOMPUTE:
//   Service classification over the last 60 days can differ from the
//   last 120 days if some pipelines haven't run recently. A service
//   that had an RA pipeline 90 days ago shouldn't lose that
//   classification just because the pipeline hasn't run within the
//   current window.
//
//   Recomputing from the rolling materialized table would make
//   classification window-dependent. Instead, the table is:
//   1. Created once (initial seed from full backfill)
//   2. Upserted daily using only NEW incoming rows
//   3. Counters only grow — never shrink
//   4. Classification is derived from cumulative counters
//
// GRAIN: One row per ServiceId
// JOIN: SDPMaterialized LEFT JOIN ServiceClassification ON ServiceId
//       (at query time in Power BI or KQL)
// ============================================================

// =============================================================
// TABLE CREATION (run once)
// =============================================================
// .create table ServiceClassification (
//     ServiceId: string,
//     TotalPipelines: long,
//     CosmicPipelines: long,
//     ClassicPipelines: long,
//     RAPipelines: long,
//     CosmicStages: long,
//     ClassicStages: long,
//     RAStages: long,
//     OnboardedPipelines: long,
//     isCosmicService: string,
//     Ev2ServiceType: string,
//     LastUpdated: datetime
// )

// =============================================================
// INITIAL SEED (run once after backfill)
// Computes from the full accumulated SDPMaterialized table
// =============================================================
// .set-or-replace ServiceClassification <|
let seed =
    SDPMaterialized
    | summarize
        HasClassic = max(HasClassic),
        HasRA = max(HasRA),
        isCosmicStage = max(isCosmicStage)
        by ServiceId, PipelineUrl
    | summarize
        TotalPipelines = dcount(PipelineUrl),
        CosmicPipelines = dcountif(PipelineUrl, isCosmicStage == 1),
        ClassicPipelines = dcountif(PipelineUrl, HasClassic == 1),
        RAPipelines = dcountif(PipelineUrl, HasRA == 1),
        CosmicStages = countif(isCosmicStage == 1),
        ClassicStages = countif(HasClassic == 1),
        RAStages = countif(HasRA == 1)
        by ServiceId
    | extend
        isCosmicService = iff(CosmicPipelines > 0, "Cosmic Service", "Entirely Non-Cosmic"),
        Ev2ServiceType = case(
            RAStages == 0 and ClassicStages > 0, "RS Only",
            RAStages > 0 and ClassicStages == 0, "RA Only",
            RAStages > 0 and ClassicStages > 0, "Hybrid Services",
            "Unknown"),
        LastUpdated = now();
seed

// =============================================================
// DAILY UPSERT (run after each daily slice is appended)
// Merges NEW day's data with existing classification
// =============================================================
//
// Strategy:
// 1. Compute classification from today's NEW rows only
// 2. Merge with existing ServiceClassification (take max of counters)
// 3. Rederive classification strings from merged counters
// 4. Replace the table
//
// Counters only grow — if a service was "Hybrid" yesterday and today's
// slice only has Classic stages, it stays "Hybrid" because RAStages > 0
// from the existing row.
//
// .set-or-replace ServiceClassification <|
let sliceStart = datetime(2026-03-30);
let sliceEnd   = datetime(2026-03-31);
// Today's new data
let todayClassification =
    SDPMaterialized
    | where InsertedAt >= sliceStart and InsertedAt < sliceEnd
    | summarize
        HasClassic = max(HasClassic),
        HasRA = max(HasRA),
        isCosmicStage = max(isCosmicStage)
        by ServiceId, PipelineUrl
    | summarize
        NewTotalPipelines = dcount(PipelineUrl),
        NewCosmicPipelines = dcountif(PipelineUrl, isCosmicStage == 1),
        NewClassicPipelines = dcountif(PipelineUrl, HasClassic == 1),
        NewRAPipelines = dcountif(PipelineUrl, HasRA == 1),
        NewCosmicStages = countif(isCosmicStage == 1),
        NewClassicStages = countif(HasClassic == 1),
        NewRAStages = countif(HasRA == 1)
        by ServiceId;
// Merge: existing + today (take max to ensure counters only grow)
let existingClassification = ServiceClassification;
let merged =
    todayClassification
    | join kind=fullouter existingClassification on ServiceId
    | extend
        ServiceId = coalesce(ServiceId, ServiceId1),
        TotalPipelines = max_of(
            coalesce(NewTotalPipelines, 0) + coalesce(TotalPipelines, 0),
            coalesce(TotalPipelines, 0)),
        CosmicPipelines = max_of(
            coalesce(NewCosmicPipelines, 0) + coalesce(CosmicPipelines, 0),
            coalesce(CosmicPipelines, 0)),
        ClassicPipelines = max_of(
            coalesce(NewClassicPipelines, 0) + coalesce(ClassicPipelines, 0),
            coalesce(ClassicPipelines, 0)),
        RAPipelines = max_of(
            coalesce(NewRAPipelines, 0) + coalesce(RAPipelines, 0),
            coalesce(RAPipelines, 0)),
        CosmicStages = max_of(
            coalesce(NewCosmicStages, 0) + coalesce(CosmicStages, 0),
            coalesce(CosmicStages, 0)),
        ClassicStages = max_of(
            coalesce(NewClassicStages, 0) + coalesce(ClassicStages, 0),
            coalesce(ClassicStages, 0)),
        RAStages = max_of(
            coalesce(NewRAStages, 0) + coalesce(RAStages, 0),
            coalesce(RAStages, 0))
    // Rederive classification from merged counters
    | extend
        isCosmicService = iff(CosmicPipelines > 0, "Cosmic Service", "Entirely Non-Cosmic"),
        Ev2ServiceType = case(
            RAStages == 0 and ClassicStages > 0, "RS Only",
            RAStages > 0 and ClassicStages == 0, "RA Only",
            RAStages > 0 and ClassicStages > 0, "Hybrid Services",
            "Unknown"),
        LastUpdated = now()
    | project
        ServiceId, TotalPipelines, CosmicPipelines, ClassicPipelines,
        RAPipelines, CosmicStages, ClassicStages, RAStages,
        OnboardedPipelines = long(0), // TODO: compute from PipelineOnboarded logic
        isCosmicService, Ev2ServiceType, LastUpdated;
merged

// =============================================================
// PIPELINE ONBOARDED (computed separately per PipelineUrl)
// A pipeline is onboarded if ALL its stages are Onboarded AND
// it has a non-empty Trainset.
// This is a pipeline-level aggregate, not service-level.
// Can be computed as a view over SDPMaterialized:
// =============================================================
// let PipelineOnboarded =
//     SDPMaterialized
//     | summarize
//         AllStagesOnboarded = min(Onboarded),
//         HasTrainset = max(iff(isnotempty(Trainset), 1, 0))
//         by PipelineUrl, ServiceId
//     | extend PipelineOnboarded = iff(AllStagesOnboarded == 1 and HasTrainset == 1, 1, 0);
//
// Join at query time:
//   SDPMaterialized | join kind=leftouter PipelineOnboarded on PipelineUrl
// =============================================================

// =============================================================
// USAGE: Join at query time (Power BI or KQL)
// =============================================================
// SDPMaterialized
// | join kind=leftouter ServiceClassification on ServiceId
// | join kind=leftouter PipelineOnboarded on PipelineUrl
// | project ..., isCosmicService, Ev2ServiceType, PipelineOnboarded
