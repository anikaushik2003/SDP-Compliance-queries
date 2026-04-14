# Backup Proposal: RawMetrics Table (Azure Data Explorer → Azure Blob Storage)

## Objective
Create a reliable backup of the RawMetrics table from Azure Data Explorer (Kusto) before using it in the test cluster.

This backup is required because:
- Production data will be copied to the test cluster for validation
- Azure Data Factory (ADF) pipelines depend on RawMetrics for backfilling derived tables in PROD
- Ensures recovery capability for SDP Compliance Dashboard data integrity

---

## Problem Summary
- Table size:
  - ~1.9M rows
  - ~2 GB compressed (on disk)
  - ~18.5 GB uncompressed
- Large payloads (~800MB+ per query) cause:
  - API failures (az rest)
  - Memory buffering issues
  - Timeouts

Client-side export is not viable.

---

## Proposed Solution
Use server-side export via Kusto `.export` to Azure Blob Storage.

### Command
```kusto
.export async compressed to csv (
    h@"https://<storage-account>.blob.core.windows.net/<container>/RawMetrics_backup?<SAS-token>"
)
with (
    sizeLimit = 1073741824,
    namePrefix = "RawMetrics",
    includeHeaders = "firstFile"
)
<|
RawMetrics
```

---

## How It Works
- Export runs inside the Kusto cluster (no client bottlenecks)
- Data is written directly to Blob Storage as compressed CSV (.gz)
- Output is split into manageable chunks (~1 GB each, since 1073741824 bytes = 1 GB)
- Entire table is exported (no truncation)

---

## Backup Output
- Format: CSV (gzip compressed)
- Estimated size: 2–3 GB total
- Storage location: Azure Blob container
- File pattern: `RawMetrics_backup_*.csv.gz`

---

## Estimated Cost (Azure Blob Storage)

### Storage Cost (Monthly)

| Tier    | Cost/GB/month | Estimated Monthly Cost (3 GB) |
|---------|--------------|-------------------------------|
| Hot     | ~$0.018      | ~$0.05                        |
| Cool    | ~$0.012      | ~$0.03                        |
| Archive | ~$0.001      | ~$0.003                       |

### Additional Cost Considerations
- Write operations: negligible for one-time export
- Read operations: minimal unless frequent restores
- Data egress: applies only if downloading outside Azure

**Recommended tier:** Cool (low cost with reasonable access latency)

---

## Restore Strategy
To restore into test or production:

```kusto
.ingest into table RawMetrics (
    h@"https://<storage-account>.blob.core.windows.net/<container>/<file>.csv.gz?<SAS>"
)
with (format='csv', ignoreFirstRecord=true)
```

---

## Requirements
- Azure Storage Account and Blob Container
- Write access (RBAC or SAS token)
- Kusto permissions to run export

---

## Key Considerations
- Backup is point-in-time
- Schema should also be captured:

```kusto
.show table RawMetrics schema as json
```

- Archive tier has higher retrieval latency
- Headers are included only in the first file (`includeHeaders = "firstFile"`)

---

## Summary
- Backup method: Kusto `.export` to Azure Blob Storage
- Size: ~2–3 GB compressed
- File chunking: ~1 GB per file
- Cost: <$0.05/month
- Purpose: Recovery, ADF backfill support, and test validation