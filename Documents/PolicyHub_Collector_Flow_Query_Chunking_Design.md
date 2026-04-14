# PolicyHub Collector Flow Query Chunking Design

| | |
|---|---|
| **Contributors** | anikaushik |
| **Reviewers** | |
| **Security & Compliance link/s** | N/A |

## Change History

| Date | Changes done | Author | Reviewers/Approvers |
|---|---|---|---|
| 26-03-2026 | Document Inception | anikaushik | |

## Table of Contents

- [Introduction](#introduction)
- [Problem Statement](#problem-statement)
  - [Goals](#goals)
- [Design Requirements](#design-requirements)
  - [Current Design](#current-design)
- [In Scope](#in-scope)
- [Out of Scope](#out-of-scope)
- [Design Proposal/s](#design-proposals)
  - [Detailed Design](#detailed-design)
  - [Components Interaction](#components-interaction)
  - [Risks and Dependencies](#risks-and-dependencies)
- [Operational Concerns](#operational-concerns)
  - [Test Strategy](#test-strategy)
  - [Telemetry and Monitoring](#telemetry-and-monitoring)
  - [Flighting & Roll-out Strategy](#flighting--roll-out-strategy)
- [Open Questions](#open-questions)
- [AI Assistance in Design](#ai-assistance-in-design)
- [References](#references)
- [Appendix](#appendix)

---

# Introduction

The PolicyHub collector flow is a component of PolicyHub Azure Function App that incrementally syncs enforcement telemetry from the M365 Gating Kusto cluster (m365gating-kusto-prod) into the central PolicyHub kusto cluster (policyHub-kusto-prod) for compliance reporting and Power BI dashboarding. It collects data from 5 kusto tables and 1 Cosmos DB collection, processing ~4.1M rows (~972 MB) across the table RuleExecutionMetrics alone over a 7-day window, with peak daily volumes of ~770k rows (RuleExecutionMetrics) and ~630k rows (PolicyVersionMetrics).

The collector uses watermark-based incremental queries, fetching only records newer than the last successful sync. This document proposes adding time-window chunking and slicing by number of rows based on Gating Response size to the collector's query strategy, replacing the current single-query-per-sync approach that causes OOM failures and silent data loss when the collector falls behind.

# Problem Statement

The current collector query pattern:

- Kusto does not provide a continuation token or server-side cursor for pagination like Cosmos DB, so the collector issues a single query for the entire time window from the last sync watermark to now.
- Issues a single Kusto query spanning the entire time window from the last sync watermark to now
- Applies `| order by [TimestampColumn] desc` which forces Kusto to sort and materialize the full result set in memory
- Appends `| take 100000` (via MaxRecords config) which silently drops rows when the window contains more than 100K records
- Does not advance the watermark on failure, causing the window to grow larger on each retry

This creates two compounding problems:

- **Kusto OOM error compounding:** When the collector fails with `E_LOW_MEMORY_CONDITION`, the watermark stays fixed while `currentSyncTime` moves forward. The next run queries an even larger window, consuming more memory, and is more likely to OOM again. A 7-day backlog on RuleExecutionMetrics produces approximately 4.1M sorted rows requiring 775 MB peak memory per node, enough to trigger OOM on a busy cluster.

- **Silent data loss:** The `| take 100000` cap drops all rows beyond 100K per query. On peak days, RuleExecutionMetrics produces ~770K rows/day and PolicyVersionMetrics ~630K rows/day, meaning up to 87% and 84% of records respectively are permanently lost each run, as the watermark advances past them.

## Goals

1. Ensure Out-of-Memory Error does not compound on failure and does not accumulate with time.
2. Break large time windows into smaller configurable chunks which are safely under kusto limits and can be adjusted as requirements change.
3. Advance watermark after each successful chunk so that failures only lose progress on current chunk, not the entire window.
4. Eliminate silent data loss.

# Design Requirements

| Requirement | Constraint |
|---|---|
| Data volume | ~4.1M rows/week for RuleExecutionMetrics and ~3.2M rows/week for PolicyVersionMetrics across 5 Kusto tables and 1 Cosmos DB collection. |
| Chunk Size | Each chunk query must stay under kusto limits by a safe margin, ~50MB peak memory per node to avoid `E_LOW_MEMORY_CONDITION` on shared clusters |
| No Data Loss | All records within the sync time window must be collected and there must not be a silent loss via `| take MaxRecords` |
| Watermark Safety | Watermarks must only advance after successful ingestion of a chunk, ensuring retry-on-failure without skipping data |
| Configurable per data source | Chunk duration must be configurable per data source, since tables have different volumes (21K/day for RawMetrics vs 770K/day for RuleExecutionMetrics) |
| Idempotency | Re-running a failed chunk must not produce duplicate rows in the destination kusto cluster |
| Execution Time | The full collection run (all chunks across all data sources) must complete within the Azure function timeout and before the next timer trigger fires. |

### Design Requirement: Time-based chunking

To prevent out-of-memory errors from compounding with time, implement fixed time-based chunks. For a backlog period of N days, when the collector flow runs, it breaks the window `[lastSyncTime, currentSyncTime]` into fixed-duration chunks (e.g., PT1H = 1 hour). Each chunk is queried, transformed, ingested, and its watermark committed independently. If the function times out after processing K chunks, the next run resumes from chunk K+1 — the system is self-healing and makes forward progress on every run.

### Design Requirement: Row-based slicing

For a given time chunk, ensure the number of selected rows are split into manageable slices based on gating response size. The constraint is:

```
avgResponseSize * rowCount < MaxMemoryThreshold
```

Where `MaxMemoryThreshold` is a configurable limit per data source. Before fetching data for a chunk, the pipeline runs a lightweight pre-query to get `count()` and `avg(estimate_data_size(*))`. If the estimated total size exceeds `MaxMemoryThreshold`, the rows are fetched in slices of size `floor(MaxMemoryThreshold / avgResponseSize)` using `| take sliceSize`. This ensures each slice stays within memory limits regardless of data volume spikes within a single time window.

## Current Design

The current collector query pattern:

- Issues a single Kusto query spanning the entire time window from the last sync watermark to now
- Applies `| order by [TimestampColumn] desc` which forces Kusto to sort and materialize the full result set in memory
- Appends `| take 100000` (via MaxRecords config) which silently drops rows when the window contains more than 100K records
- Does not advance the watermark on failure, causing the window to grow larger on each retry

![current pipeline flow](./current_pipeline_flow.png)
**Measured impact of current design (prod cluster, RuleExecutionMetrics):**

| Window Size | Rows | Data Size | Peak Memory/Node | Exec Time |
|---|---|---|---|---|
| 7 days | 4,095,471 | ~972 MB | 775 MB | 0.55s |
| 1 day | 768,826 | ~182 MB | 175 MB | 0.06s |
| 6 hours | 193,624 | ~45 MB | 51 MB | 0.007s |
| 1 hour | 46,503 | ~11 MB | 16 MB | ~0s |
| 30 minutes | 21,464 | ~5 MB | 13 MB | ~0s |

# In Scope

- Dealing with Kusto Low memory condition error
- Ensuring Collector Flow design scales and can accommodate an increase in the gating response size
- Backfilling data since last successful collector flow run.

# Out of Scope

- Syncing of PolicyAssignments is OOS as this will be handled in the same way as Policy Sync service pagination via continuation token.

# Design Proposal/s

## Detailed Design

### Overview

The solution introduces two layers of chunking to the collector pipeline:

1. **Time-window chunking** (outer loop): Breaks the `[lastSyncTime, currentSyncTime]` range into fixed-duration chunks configured via `ChunkDuration` (ISO 8601 duration, e.g., `"PT1H"`). Each chunk is processed independently with its own watermark commit.

2. **Row-based slicing** (inner loop): Within a single time chunk, a pre-query determines `rowCount` and `avgResponseSize`. If `avgResponseSize * rowCount > MaxMemoryThreshold`, the chunk's rows are fetched in slices of `floor(MaxMemoryThreshold / avgResponseSize)` rows each. This handles anomalous spikes where a single hour may contain disproportionate data that exceeds memory limits even within a short time window.

### Chunked Collection Flow

![Chunked Collection Flow Diagram](./Collector_Pipeline_Flow_New.png)

### Chunk Duration Selection

Based on smoke testing against the prod M365 Gating cluster:

| ChunkDuration | Rows/chunk (RuleExecutionMetrics) | Peak Memory/Node | Recommendation |
|---|---|---|---|
| PT30M | ~21K | ~13 MB | Safest, more Cosmos watermark writes |
| **PT1H** | **~47K** | **~16 MB** | **Recommended — safe margin, reasonable chunk count** |
| PT6H | ~194K | ~51 MB | Acceptable for smaller tables |
| PT1D | ~770K | ~175 MB | Risky on busy clusters |

**Recommended default: `PT1H`** for all data sources. Larger tables (RuleExecutionMetrics, PolicyVersionMetrics) stay well under memory limits. Smaller tables (RawMetrics, GatingRequestMetrics) produce near-empty chunks which are skipped efficiently.

### Configuration Model

`ChunkDuration` and `MaxMemoryThreshold` are added to the per-data-source `QueryConfig`:

```json
{
    "Name": "RuleExecutionMetrics",
    "Type": "Kusto",
    "IsEnabled": true,
    "Connection": { ... },
    "Query": {
        "QueryText": "RuleExecutionMetrics | order by InsertedAt desc",
        "TimestampColumn": "InsertedAt",
        "ChunkDuration": "PT1H",
        "MaxMemoryThreshold": 52428800
    },
    "DestinationTable": { ... }
}
```

- **`ChunkDuration`** (nullable string, ISO 8601 duration): When set, the pipeline chunks the time window. When null/omitted, the pipeline executes a single query over the full window (backward compatible).
- **`MaxMemoryThreshold`** (nullable long, bytes): Maximum estimated data size per query. When set, the pipeline runs a pre-query for each chunk to get `count()` and `avg(estimate_data_size(*))`. If `avgSize * rowCount > MaxMemoryThreshold`, the chunk is fetched in row-based slices of `floor(MaxMemoryThreshold / avgSize)` rows each. When null, no row-based slicing is performed (time chunking alone controls query size). Example: `52428800` = 50 MB.
- **`MaxRecords`** is removed from Kusto query generation. The `| take N` clause is no longer appended to queries — time-window chunking and row-based slicing replace it as the mechanism for bounding query size.

### Files Modified

| File | Change |
|---|---|
| `Models/FunctionAppSettings/CollectorConfigs/ICollectorConfigs.cs` | Added `ChunkDuration` property to `IQueryConfig` interface |
| `Models/FunctionAppSettings/CollectorConfigs/CollectorConfigs.cs` | Added `ChunkDuration` property to `QueryConfig` class |
| `Services/Collector/Pipelines/CollectorPipeline.cs` | Added chunking loop with per-chunk watermark advancement and `GetChunkDuration()` helper |
| `Services/Collector/QueryBuilder/KustoQueryBuilderService.cs` | Removed `MaxRecords` / `| take N` limit clause |

### Backward Compatibility

When `ChunkDuration` is null (not configured), `GetChunkDuration()` returns `currentSyncTime - lastSyncTime` (the full window). The `while` loop executes exactly once, producing identical behavior to the pre-change code. No existing data sources are affected until `ChunkDuration` is explicitly set.

### Backlog Catch-Up Behavior

For a large backlog (e.g., 120 days), the system self-heals across multiple runs:

| Run | Backlog | Chunks processed (example) | Remaining |
|---|---|---|---|
| 1 | 120 days | 576 chunks (24 days) | 96 days |
| 2 | 96 days | 576 chunks (24 days) | 72 days |
| 3 | 72 days | 576 chunks (24 days) | 48 days |
| ... | ... | ... | Caught up |

Each run makes forward progress regardless of whether it completes the full backlog. The watermark advances after each successful chunk, so no work is repeated.

### Idempotency

The destination Kusto tables use queued ingestion. If a chunk is ingested but the watermark update fails (e.g., Cosmos timeout), the next run re-queries the same time window and re-ingests. Kusto's append-only model means duplicate rows may appear. This is the existing behavior and is mitigated by:
- Kusto's deduplication policies on the destination tables (if configured)
- The narrow chunk window (1 hour) limits the scope of any duplication
- Downstream Power BI queries can deduplicate via `arg_max()` or `distinct`

## Components Interaction

### Data Flow

![Data Flow Diagram Placeholder](./PolicyHub_data_collection_flow.png)

### Module Interaction

| Module | Role | Interaction |
|---|---|---|
| `TimerDataCollectorFunction` / `HttpDataCollectorFunction` | Trigger | Invokes `CollectorWorkflow.ExecuteCollectionWorkflowAsync()` |
| `CollectorWorkflow` | Orchestrator | Iterates enabled systems in parallel, calls `CollectorPipeline` per data source (sequential within a system) |
| `CollectorPipeline` | **Chunking loop (modified)** | Manages time-window iteration, calls provider per chunk, commits watermark per chunk |
| `KustoDataSourceProvider` | Data fetch | Executes KQL query for a single chunk time range, returns `DataTable` |
| `KustoQueryBuilderService` | **Query builder (modified)** | Builds `{baseQuery} \| where {ts} > chunkStart and {ts} <= chunkEnd` (no `\| take`) |
| `DataTransformationService` | Transform | Converts `DataTable` rows to JSON strings |
| `DataIngestionService` | Ingest | Parallel ingestion to destination Kusto cluster |
| `SyncStateManagerService` | Watermark | Reads/writes `CollectorSyncState` in Cosmos DB per chunk |
| `ICollectorConfigs` / `QueryConfig` | **Config (modified)** | Provides `ChunkDuration` per data source |

### Data Contract — `CollectorSyncState` (unchanged schema)

```json
{
    "id": "{enforcerName}_{clusterName}_{databaseName}_{tableName}",
    "enforcerName": "M365Gating",
    "lastSyncTimestamp": "2026-03-27T14:00:00Z",
    "lastSyncStatus": "Success | Failed | Skipped",
    "lastSyncRecordCount": 46503,
    "lastSyncError": null
}
```

**Behavior change**: `lastSyncTimestamp` now advances per chunk (e.g., in 1-hour increments) rather than jumping to `currentSyncTime` at the end of the full run. The watermark only advances after **all row slices** within a chunk are successfully ingested. If any slice fails mid-chunk, the watermark stays at the previous chunk boundary and the entire chunk is retried on the next run. This keeps the chunk as the atomic unit of progress — simpler state management, and the bounded chunk size means re-ingesting a few slices on retry is low-cost.

### Data Contract — `QueryConfig` (modified)

```json
{
    "QueryText": "RuleExecutionMetrics | order by InsertedAt desc",
    "TimestampColumn": "InsertedAt",
    "ChunkDuration": "PT1H",
    "MaxMemoryThreshold": 52428800,
    "Parameters": {}
}
```

| Field | Type | Description |
|---|---|---|
| `QueryText` | string (required) | Base KQL query |
| `TimestampColumn` | string (required) | Column used for incremental filtering |
| `ChunkDuration` | string? (new) | ISO 8601 duration (e.g., `"PT1H"`, `"PT30M"`). Null = full window (backward compatible). |
| `MaxMemoryThreshold` | long? (new) | Max estimated data size in bytes per query (e.g., `52428800` = 50 MB). When set, enables row-based slicing within each time chunk. Null = no row slicing. |
| `MaxRecords` | int? (existing, now ignored for Kusto) | No longer appended as `\| take` for Kusto queries. Retained in config model for Cosmos `PageSize` usage. |

## Risks and Dependencies

| Risk | Mitigation |
|---|---|
| **Large backlog exceeds function timeout** | Self-healing: each run makes partial progress. Watermark advances per chunk, next run resumes where the last left off. For initial catch-up, use HTTP trigger (longer timeout) or temporarily increase `ChunkDuration`. |
| **Duplicate rows on retry** | Existing behavior, not introduced by this change. Mitigated by destination deduplication policies and narrow chunk windows. |
| **Kusto source retention expiry** | If the source cluster's retention policy drops data before the collector catches up, that data is permanently lost. Monitor watermark lag and alert when it approaches retention period. |
| **Abnormal spike within a single chunk** | A single hour with 10x normal volume could approach memory limits. Row-based slicing via `MaxMemoryThreshold` addresses this — the pipeline pre-queries to estimate data size and automatically slices when the threshold is exceeded. As a secondary control, `ChunkDuration` can be reduced to `PT30M` or `PT15M` for affected data sources. |
| **Cosmos DB RU cost for watermark writes** | One Cosmos write per non-empty chunk instead of one per run. At PT1H with 5 data sources, worst case ~120 writes/day. Negligible RU impact. |
| **Dependency on PR 5027618** | The `QueryDocumentsPageAsync` method added by PR 5027618 is required for future Cosmos pagination (out of scope). Time-window chunking has no dependency on this PR. |

# Operational Concerns

## Test Strategy

### Unit Tests

- Existing 651 unit tests pass with no modifications — backward compatibility confirmed.
- `ChunkDuration = null` produces single-iteration loop identical to pre-change behavior.

### Integration / Smoke Testing

Performed read-only Kusto queries against `m365gating-kusto-prod/M365GatingAnalytics` to validate:

1. **Data volume per chunk size**: Counted rows and measured `estimate_data_size(*)` for PT30M, PT1H, PT6H, PT1D, and 7-day windows.
2. **Memory per chunk size**: Measured `peak_per_node` memory via Kusto query completion stats with `| order by` + `| serialize` to force sort materialization.
3. **Confirmed PT1H produces ~47K rows / ~16 MB peak memory** for the largest table (RuleExecutionMetrics) — well within safe Kusto limits.

### Validation Plan

1. Deploy to test environment with `ChunkDuration: "PT1H"` configured for all Kusto data sources.
2. Trigger via HTTP endpoint (`GET /data-collection`).
3. Verify in logs: chunk iteration count, rows per chunk, watermark advancement.
4. Verify in destination Kusto: row counts match source cluster for the synced time window.
5. Simulate failure mid-run (disable function after N chunks) and verify next run resumes from last committed watermark.

## Telemetry and Monitoring

### Current Telemetry and Monitoring

The collector flow currently has the following telemetry:

**Application Insights Metrics (via `MetricTracker` wrapping `TelemetryClient`):**
- **`PolicyHubLatency`**: Execution time in ms for the entire `ExecuteDataCollectionAsync` call, with dimensions `Event` (method name) and `IsSucceeded`. Emitted once per collector run by `BaseDataCollectorFunction`.
- **`PolicyHubStatus`**: Success/failure count with dimensions `Event`, `Status`, `ErrorType`, `ErrorCode`. Emitted once per collector run.

**Azure Monitor Alert Rules (deployed via Ev2 Bicep):**
- **Latency alert**: Fires when average latency of `ExecuteDataCollectionAsync` exceeds 10,000 ms over PT2H window. Currently **disabled**.
- **Cron failure alert**: Fires when success count of `ExecuteDataCollectionAsync` drops below 1 in a PT6H window. **Enabled**, routes to IcM via `Policyhub://AzureMonitor` routing ID.
- **System error alert**: Fires when `ErrorType == "SystemErrorException"` count > 0 in PT2H. **Enabled**, routes to IcM.
- **Availability alert**: Azure-native health check on the Function App, fires when `HealthCheckStatus < 90%` over PT30M. **Enabled**, routes to IcM.

**Structured Logging (via `ICustomLogger`):**
- All collector components (`CollectorPipeline`, `CollectorWorkflow`, `KustoDataSourceProvider`, `SyncStateManagerService`, etc.) log to Application Insights traces.

**Watermark State (Cosmos DB):**
- `CollectorSyncState` documents in Cosmos DB contain `lastSyncTimestamp`, `lastSyncStatus`, `lastSyncRecordCount`, and `lastSyncError` per data source. Not actively monitored — must be queried manually.

### Why Current Monitoring Is Insufficient

The current monitoring was designed for a single-query-per-run model and does not account for the chunked pipeline or the failure modes that caused the OOM issue:

1. **No watermark lag visibility.** The cron failure alert only fires if the function itself fails. If the collector runs successfully but processes stale data (e.g., watermark is 5 days behind), there is no alert. The OOM death spiral was not caught because each run was "succeeding" in the sense that the function executed — it just failed silently by dropping data via `| take 100000`.

2. **No per-chunk telemetry.** The current metrics track the entire `ExecuteDataCollectionAsync` call as a single unit. With chunking, a run could process 20 chunks successfully then timeout on chunk 21, but it would be reported as a single failure. There is no visibility into partial progress, which chunk failed, or why.

3. **No error classification for RCA.** The `ErrorType` and `ErrorCode` dimensions exist but are generic (`SystemErrorException`). An on-call engineer cannot distinguish between a Kusto OOM, a Cosmos timeout, an auth failure, or an ingestion error without reading raw logs. Each of these has a different RCA path and remediation.

4. **No resource utilization tracking.** There is no visibility into Kusto query memory consumption, execution time per query, or how close queries are to cluster limits. Without this, the team cannot proactively detect when data growth is approaching chunk size thresholds.

5. **No end-to-end latency metric.** There is no measure of the time between when a gating metric is created at the source (`MetricCreationTime` / `InsertedAt`) and when it becomes available in the destination PolicyHub Kusto cluster. This is the metric that matters to dashboard consumers — they care about data staleness, not function execution time.

6. **No consecutive failure tracking.** A single transient failure is different from 10 consecutive failures on the same data source. The current alerting treats them identically. An on-call engineer needs to quickly distinguish a transient blip from a systematic breakdown.

### Proposed Additions

#### 1. Data Freshness / Watermark Lag

Tracks how far behind the pipeline is per data source.

- **Metric**: `CollectorWatermarkLagSeconds` — `currentTime - lastSyncTimestamp` per data source
- **Dimensions**: `SystemName`, `DataSourceName`, `DataSourceType`
- **Source**: Computed from `CollectorSyncState` in Cosmos DB after each run
- **Alert**: Fire to IcM when lag exceeds 2x the cron interval (>12 hours for a 6-hour cron). A second, higher-severity alert when lag approaches the source Kusto cluster's retention period (data loss imminent).

#### 2. Per-Chunk Metrics

Enables visibility into partial progress and identifies which specific chunks are problematic.

- **Metric**: `CollectorChunkLatency` — execution time per chunk (query + transform + ingest)
- **Dimensions**: `SystemName`, `DataSourceName`, `ChunkIndex`, `ChunkStart`, `ChunkEnd`, `RowCount`, `IsSucceeded`
- **Metric**: `CollectorChunksProcessed` — count of successfully processed chunks per run
- **Dimensions**: `SystemName`, `DataSourceName`, `TotalChunks`, `SuccessfulChunks`, `FailedChunks`

#### 3. Error Classification

Categorizes failures to enable faster root cause identification without reading raw logs.

- **Metric**: `CollectorErrorClassification` — categorized error type per failure
- **Dimensions**: `SystemName`, `DataSourceName`, `ErrorCategory`, `ErrorDetail`
- **Error categories**:

| ErrorCategory | Meaning | Remediation |
|---|---|---|
| `KustoOOM` | `E_LOW_MEMORY_CONDITION` from Kusto | Data volume spike — reduce `ChunkDuration` or increase `MaxMemoryThreshold` |
| `KustoTimeout` | Query exceeded server timeout | Query too broad or cluster under load — check cluster health |
| `KustoAuthFailure` | 401/403 from Kusto endpoint | MSI credential rotation or RBAC change — check identity permissions |
| `CosmosTimeout` | Cosmos DB request timeout | RU throttling or partition hotspot — check Cosmos metrics |
| `CosmosAuthFailure` | 401/403 from Cosmos | MSI or connection string issue |
| `IngestionFailure` | Destination Kusto ingestion failed | Destination cluster issue — check schema mismatch, cluster health |
| `TransformationError` | Data transformation failed | Schema drift in source data — check source table schema |
| `WatermarkUpdateFailure` | Failed to update sync state in Cosmos | Cosmos RU or connectivity — check Cosmos health |
| `ConfigurationError` | Invalid `ChunkDuration`, missing config | Deployment issue — check appsettings |

- **Alert**: Fire to IcM on any `KustoOOM` or 3+ consecutive failures of any category on the same data source.

#### 4. Resource Utilization

Tracks how close queries are to cluster limits. Enables proactive resizing before failures occur and distinguishes between data volume spikes (sudden) and organic growth (gradual trend).

- **Metric**: `CollectorQueryResourceUsage` — Kusto query stats per chunk
- **Dimensions**: `SystemName`, `DataSourceName`, `PeakMemoryBytes`, `ExecutionTimeMs`, `RowCount`, `DataSizeBytes`, `ChunkDuration`
- **Source**: Extracted from Kusto query completion stats (available in the v2 REST API response `QueryCompletionInformation` frame — fields `resource_usage.memory.peak_per_node` and `ExecutionTime`)
- **Alert**: Fire warning when `PeakMemoryBytes` exceeds 70% of `MaxMemoryThreshold` for any data source, indicating that current chunk sizing is approaching limits.
- **Dashboard**: Trend `PeakMemoryBytes` and `RowCount` per data source over time. An upward trend means gating response sizes or volumes are growing and chunk configuration may need adjustment. A sudden spike correlated with an incident suggests a temporary condition. A flat memory trend with increasing failures suggests an infrastructure problem rather than a data volume problem.

#### 5. End-to-End Latency (Event Time to Ingestion Time)

Measures data staleness from the perspective of downstream consumers (Power BI dashboards). This is distinct from function execution latency — a function can execute quickly but still be processing data that is days old.

- **Metric**: `CollectorEndToEndLatencySeconds` — `ingestionTime - avg(eventTime)` per chunk
- **Dimensions**: `SystemName`, `DataSourceName`, `ChunkStart`, `ChunkEnd`
- **Source**: `eventTime` is the `TimestampColumn` value (e.g., `MetricCreationTime`, `InsertedAt`) from the source data. `ingestionTime` is captured when the chunk is successfully ingested. The difference represents how long it takes for a gating metric to flow from the enforcer to the PolicyHub dashboard.
- **Alert**: Fire when end-to-end latency exceeds a threshold (e.g., >24 hours). This enables SLA tracking for dashboard consumers — "compliance data is available within X hours of generation."
- **Dashboard**: During backlog catch-up, end-to-end latency will be high but decreasing — this is expected. During steady state, a sudden increase indicates the pipeline is falling behind or ingestion is slowing. Comparing this metric with watermark lag distinguishes "pipeline is slow" from "pipeline is behind."

#### 6. Consecutive Failure Tracking

Distinguishes transient failures from systematic breakdowns. A single failure followed by recovery is a blip; 5+ consecutive failures on the same data source with the same error category indicates a stuck pipeline that requires intervention.

- **Metric**: `CollectorConsecutiveFailures` — rolling count of consecutive failed runs per data source
- **Dimensions**: `SystemName`, `DataSourceName`, `LastErrorCategory`
- **Source**: Derived from `CollectorSyncState.lastSyncStatus`. Increment on failure, reset to 0 on success.
- **Alert**: Fire at 3 consecutive failures (transient issues should self-heal within 1-2 retries). Escalate severity at 5+ consecutive failures.

### Proposed Metrics Summary

| Metric | Purpose | Key Dimensions | Alert Condition |
|---|---|---|---|
| `CollectorWatermarkLagSeconds` | Pipeline freshness | SystemName, DataSourceName | Lag > 12h (warn), lag approaching retention period (critical) |
| `CollectorChunkLatency` | Per-chunk execution time | SystemName, DataSourceName, ChunkIndex, RowCount | — |
| `CollectorChunksProcessed` | Partial progress visibility | SystemName, TotalChunks, SuccessfulChunks, FailedChunks | FailedChunks > 0 |
| `CollectorErrorClassification` | Failure categorization | SystemName, DataSourceName, ErrorCategory | Any `KustoOOM`; 3+ consecutive failures |
| `CollectorQueryResourceUsage` | Proactive capacity tracking | SystemName, PeakMemoryBytes, RowCount, DataSizeBytes | PeakMemory > 70% of MaxMemoryThreshold |
| `CollectorEndToEndLatencySeconds` | Data staleness for consumers | SystemName, DataSourceName, ChunkStart | Latency > 24h |
| `CollectorConsecutiveFailures` | Transient vs systematic failure | SystemName, DataSourceName, LastErrorCategory | Count >= 3 (warn), >= 5 (critical) |

## Flighting & Roll-out Strategy

### Deployment Plan

1. **Stage 1 — Test environment**: Deploy with `ChunkDuration: "PT1H"` for all Kusto data sources. Validate via HTTP trigger.
2. **Stage 2 — PPE environment**: Deploy and monitor for 1-2 days. Verify watermark advancement and data completeness.
3. **Stage 3 — Prod environment**: Deploy with `ChunkDuration: "PT1H"`. Monitor initial backlog catch-up via watermark lag dashboard.

### Rollback Plan

- **Config rollback**: Remove `ChunkDuration` from appsettings to revert to single-query behavior (but re-introduces OOM risk).
- **Code rollback**: Revert the 4 modified files. No database migrations, schema changes, or external contract changes to unwind.
- **Feature flag**: `EnableTimerDataCollectorFunction` / `EnableHttpDataCollectorFunction` can disable the collector entirely if needed.

# Open Questions

1. What is the Kusto retention policy on `m365gating-kusto-prod`? This determines the maximum backlog that can be recovered.
2. What alerting threshold should be set for watermark lag (e.g., alert if >6 hours behind)?
3. What is the appropriate default `MaxMemoryThreshold` value? Based on smoke testing, 50 MB (~51 MB peak for 6h window) provides a safe margin. Should this be standardized or tuned per data source?

# AI Assistance in Design

- Used Claude Code (Opus 4.6) to explore and understand the full PolicyHub collector codebase across 22+ source files.
- Claude performed read-only smoke testing against the prod Kusto cluster, executing count and memory-measurement queries for 5 different time-window sizes to determine optimal chunk duration.
- Claude implemented the time-window chunking code changes (4 files modified) and validated with build + 651 passing tests.
- Claude assisted in drafting sections of this design document based on codebase analysis and smoke test results.

# References

- PR 5027618: [Fix OutOfMemoryException in PolicySyncWorkflow](https://dev.azure.com/O365Exchange/O365%20Core/_git/PolicyHub/pullrequest/5027618) — pagination pattern for Cosmos DB in the sync service
- PolicyHub Repository: https://o365exchange.visualstudio.com/O365%20Core/_git/PolicyHub
- Kusto `E_LOW_MEMORY_CONDITION` documentation: Error code `80DA0007`

# Appendix

### Alternative Considered: MaxRecords with Offset Pagination

Instead of time-window chunking, paginate via `| take N | skip M` patterns in Kusto. **Rejected** because:
- Kusto does not support `OFFSET` / cursor-based pagination natively
- `| take N` after `| order by` still requires full materialization before truncation — does not reduce memory
- Time-window chunking naturally aligns with the existing watermark-based incremental sync model

### Alternative Considered: Timestamp-Cursor Row Pagination

Within a time chunk, paginate rows by using the timestamp of the last processed row as a cursor: `| where {ts} > lastProcessedTimestamp and {ts} <= chunkEnd | take sliceSize`. After each slice, advance the cursor to the last row's timestamp. **Rejected** because:
- Timestamps are not unique identifiers — multiple rows can share the same timestamp value
- Rows with the same timestamp as the cursor would be either duplicated (if using `>=`) or dropped (if using `>`)
- The pre-query approach (Option C) avoids this entirely by computing `sliceSize` from `avg(estimate_data_size(*))` and fetching deterministic slices without needing a cursor

### Alternative Considered: Reduce MaxRecords and Increase Timer Frequency

Run the collector every 5 minutes with `MaxRecords: 50000` to keep each query small. **Rejected** because:
- Still silently drops data during spikes or outages
- Does not address the compounding failure problem (watermark stays fixed)
- Fragile — requires careful tuning of timer frequency vs data volume, breaks when volumes change
