# SDP Compliance Design Document — Architecture Review Transcript

**Document under review:** `SDP-Cumulative-design-document.md`
**Review initiated:** 2026-03-03
**Team Lead:** Claude Code (Orchestrator)

---

## Round 1

### Reviewer (Senior Software Engineer) — Findings

The Reviewer completed a thorough 10-point analysis covering failure modes, data consistency, incremental correctness, cross-midnight runs, deduplication, NULL-heavy rows, missing columns, belowarmpipelines computation, ADX "atomic merge" semantics, and schema-vs-query gaps.

**Summary of Required Actions:**

| # | Finding | Severity | Action Required |
|---|---------|----------|-----------------|
| 5 | `leftanti` prevents UPSERT; sliding window is non-functional | **CRITICAL** | Replace with delete-then-append or extent-tag-based replace |
| 3b | `exemptedPipelines` filter not applied | **HIGH** | Apply filter in all-stage-runs or final join |
| 4 | No RunDate anchoring for cross-midnight runs | **HIGH** | Implement canonical RunDate per BuildId |
| 7 | Missing ServiceTree columns (Workload, DevOwner, etc.) | **HIGH** | Add ServiceTree enrichment step |
| 10e | Queries use `now()` despite design requiring parameterized dates | **HIGH** | Parameterize all date ranges |
| 1 | No rollback strategy for partial sub-query failures | **HIGH** | Add run-level staging isolation |
| 2 | Sliding window size contradiction (2-day vs 3-day) | **MEDIUM** | Resolve and document |
| 6 | No DataCompleteness indicator for NULL-heavy rows | **MEDIUM** | Add TelemetryPresent column |
| 8 | `belowarmpipelines` recomputed on every run (full table scan) | **MEDIUM** | Extract to reference table |
| 9 | "Atomic merge" terminology misleading for ADX | **MEDIUM** | Clarify ADX-specific implementation |
| 10c | query.md stage-telemetry is stale (missing MOBR/Cosmic) | **MEDIUM** | Synchronize with standalone file |
| 10f | HasLockbox source change (YAML→TimelineRecords) undocumented | **MEDIUM** | Document and validate equivalence |
| 3c | Dead code in all sub-queries | **LOW** | Clean up unused let-bindings |

**Key Architectural Risks Identified:**
1. ServiceTree snapshot drift — not persisting metadata at ingestion time
2. Policy data retention cliff — no mechanism to distinguish "retention-limited" vs "not yet available"
3. Concurrent backfill safety — parallel ADF runs could collide on delete-then-append
4. stageDataQuery.md is orphaned — superseded but still in repository

**Full detailed findings:** See Reviewer agent output above (10 sections with specific line-number references and required changes per section).

---

### Fact Checker (Product Manager) — Findings

**Claim Verification Table:**

| # | Claim | Verdict | Key Notes |
|---|---|---|---|
| 1 | ADX join memory budget 5 GB default, 30 GB max | **Correct** | Confirmed in docs |
| 2 | Cross-cluster subquery truncation 64 MB, cannot be bypassed | **Misleading** | Request-limits-policy doc says truncation limits "don't apply to intermediate results of subqueries, such as those that result from having cross-cluster references." Contradicts query-limits page. |
| 3 | Interactive query timeout 4 min default, 1 hr max | **Correct** | Confirmed |
| 4 | `set notruncation` and `set norequesttimeout` valid | **Correct** | `norequesttimeout` is technically a boolean property, not standalone `set` |
| 5 | Error 80DA0001 = join memory budget | **Correct** | Confirmed |
| 6 | `.set-or-append` avoids client timeout | **Correct** | Request limits turned off for ingest-from-query in default workload group. Recommend <1GB per operation. |
| 7 | `join kind=leftanti` valid for dedup | **Correct** | Documented join flavor |
| 8 | Update Policies can't do global reconciliation | **Correct** | Policy only sees newly ingested data from single source |
| 9 | Continuous Export can't write to ADX table | **Correct** | Only exports to external tables (Blob, ADLS) |
| 10 | BuildId belongs to exactly one day | **Unverified** | Domain-specific, not an ADX claim. Build started 23:59 could complete next day. |
| 11 | 2-day sliding window for late data | **Unverified** | Application design choice. Needs upstream SLA data to validate. |
| 12 | Staging + atomic merge prevents partial writes | **Correct** | ADX has `.replace extents`, `.update table`, `.move extents`, `.rename tables`. `.set-or-append` alone does NOT prevent partial visibility. |
| 13 | `maxmemoryconsumptionperiterator` = 30 GB | **Correct** | Also capped at 50% of node RAM on small SKUs |
| 14 | 220K rows per chunk within limits | **Misleading** | Row count insufficient — need byte-size estimate. Docs recommend <1GB per `.set-or-append`. |

**Most Critical Finding:** Claim 2 — the document's premise that cross-cluster truncation "cannot be bypassed" is contradicted by Microsoft's request-limits-policy doc. If the exclusion holds, the motivation for materialization may be overstated. However, the two doc pages contradict each other, so empirical testing is recommended.

**All citations provided** from learn.microsoft.com (query-limits, request-limits-policy, ingest-from-query, set-timeout-limits, leftanti-join, update-policy, continuous-data-export, replace-extents).

---

### Team Lead — Round 1 Synthesis

**Combined findings from Fact Checker + Reviewer:**

**CRITICAL (1):**
- Reviewer #5: `leftanti` dedup prevents UPSERT — sliding window is non-functional

**HIGH (6):**
- Reviewer #3b: `exemptedPipelines` filter is dead code (functional regression)
- Reviewer #4: No RunDate anchoring for cross-midnight runs
- Reviewer #7: Missing ServiceTree columns (Workload, DevOwner, etc.)
- Reviewer #10e: Queries use `now()` despite design requiring parameterized dates
- Reviewer #1: No rollback strategy for partial sub-query failures
- Fact Checker #2: Cross-cluster truncation claim is misleading — need to soften language or cite contradictory docs

**MEDIUM (7):**
- Reviewer #2: Sliding window 2d vs 3d contradiction
- Reviewer #6: No DataCompleteness/TelemetryPresent indicator
- Reviewer #8: belowarmpipelines full-table scan on every run
- Reviewer #9: "Atomic merge" misleading terminology for ADX
- Reviewer #10c: query.md stale vs standalone files
- Reviewer #10f: HasLockbox source change undocumented
- Fact Checker #14: 220K rows claim needs byte-size estimate

**UNVERIFIED (2):**
- Fact Checker #10: BuildId-to-day mapping
- Fact Checker #11: 2-day window justification needs upstream SLAs

**Decision:** Proceeding to Engineer phase. The Reviewer's CRITICAL finding (#5 leftanti) and the Fact Checker's misleading claim (#2 cross-cluster truncation) are the highest priority corrections.

---

### Phase 2: Reviewer Reaction to Fact Checker Findings

**Q1: Does the cross-cluster finding change any of the 10 findings?**
**No.** All 13 findings are about the new design's internal correctness — they apply regardless of the materialization motivation. The cross-cluster claim appears only in the Problem Statement as one of three constraints. No findings reference or depend on it.

**Q2: Should the Problem Statement be rewritten?**
**Yes, but carefully.** Specific recommendations:

1. **Rewrite, don't remove** the cross-cluster row in Section 2 table. Change "cannot be bypassed" to: "May apply to remote calls depending on query shape; mitigated by local snapshot tables (see optimizations 3-4)."
2. **Keep the conclusion** (line 49) as-is — it correctly cites join memory + timeout as the binding constraints.
3. **Reword Goal #3** from "Eliminate cross-cluster join bottlenecks" to "Use local snapshot tables to avoid cross-cluster overhead and reduce query complexity."

**Bottom line:** The materialization architecture is well-justified by join memory budget and timeout alone. Cross-cluster truncation is a supporting detail, not a load-bearing argument. Correcting it strengthens credibility.

---

### Phase 3: Engineer Corrections — Completed

All 12 corrections applied to `SDP-Cumulative-design-document.md`:

| # | Correction | Type | Status |
|---|---|---|---|
| 1 | Replaced `leftanti` with delete-then-append | CRITICAL | Applied |
| 2 | Softened cross-cluster truncation claim | HIGH | Applied |
| 3 | Added ServiceTree enrichment (Step 3) + materialized table columns | HIGH | Applied |
| 4 | Added exemptedPipelines note to Table 1 | HIGH | Applied |
| 5 | Parameterized dates already documented | HIGH | No change needed |
| 6 | Added RunDate concept for cross-midnight runs | HIGH | Applied |
| 7 | Added rollback/cleanup strategy | HIGH | Applied |
| 8 | Resolved sliding window to 2-day with rationale | MEDIUM | Applied |
| 9 | Added TelemetryPresent column | MEDIUM | Applied |
| 10 | Fixed "atomic merge" → "delete-then-append (idempotent replace)" | MEDIUM | Applied |
| 11 | Added byte-size estimate (165 MB/chunk) | MEDIUM | Applied |
| 12 | Documented HasLockbox source change (YAML→TimelineRecords) | MEDIUM | Applied |

**Additional fix by Team Lead:** Goal #3 reworded per Reviewer guidance: "Eliminate cross-cluster join bottlenecks" → "Use local snapshot tables to avoid cross-cluster overhead and reduce query complexity"

---

## Round 2 — Verification

### Reviewer — Round 2
**Verdict: "No Further Technical Issues."**
All CRITICAL and HIGH findings correctly addressed. Remaining items (query.md sync, dead code cleanup, belowarmpipelines implementation detail) are implementation tasks, not design document deficiencies.

Minor suggestion (non-blocking): Add one sentence about the query-visible gap during delete-then-append: "Between the delete and append, dashboard queries will see zero rows for the affected RunDate window. The 04:00 UTC schedule minimizes user impact."

### Fact Checker — Round 2
**Verdict: "No Further Factual Issues."**
- Cross-cluster claim (Finding #2): Resolved — appropriately hedged language
- Byte-size estimate (Finding #14): Resolved — 165 MB substantiates the claim
- BuildId-to-day (Finding #10): Now supported by RunDate design guarantee
- 2-day window (Finding #11): Now justified with honest empirical basis
- All remaining claims verified correct or adequately supported

---

## Phase 2: Stakeholder Review

### Stakeholder (CVP) — Findings

**Evaluation of specific design choices:**

| Design Choice | Verdict | Rationale |
|---|---|---|
| 3-table decomposition | Acceptable | Table 2 (yaml-to-run-list) could be inlined, but separation aids debuggability. Minimal overhead. |
| RunDate concept | Keep | Solves a real correctness problem. Implementation is one KQL line. Removing it creates a subtle data quality bug. |
| Staging table + cleanup | Proportional | Negligible cost (3 `drop table` commands), real risk mitigation for ADX's lack of transactions. |
| Weekly service classification recompute | Acceptable | Complexity comes from the domain, not over-engineering. Weekly cadence is reasonable. |
| Full replace daily vs incremental | **Open question** | If each sub-query can handle 60 days, full replace is strictly simpler and should be the default. |

**Cost assessment: Low** — no new infrastructure, ~570 MB storage, one daily ADF trigger.
**Implementation time: Reasonable** — sub-queries already written, ADF is straightforward.
**Operational burden: Low** — one retry policy, freshness monitoring, manual backfill for 2+ day failures.
**Maintainability: Good** — KQL stays KQL, modular sub-queries, well-documented schema.

**Verdict: "Acceptable cost, acceptable complexity, maintainable design"** — with one conditional:

> Can each of the 3 decomposed sub-queries execute on the full 60-day window without timing out? If yes, the design should include a "full replace" mode as the default (eliminating sliding window, RunDate, delete-then-append, late-data handling), with incremental mode as fallback only if full replace proves too slow.

---

## Review Completion Status

| Role | Round 1 | Round 2 | Final Verdict |
|---|---|---|---|
| Fact Checker | 8 correct, 2 misleading, 2 unverified | All resolved | **No Further Factual Issues** |
| Reviewer | 1 critical, 5 high, 7 medium | All addressed | **No Further Technical Issues** |
| Engineer | Applied 12 corrections | Verified by above | Complete |
| Stakeholder | Cost/complexity review | N/A | **Acceptable** (conditional on full-replace question) |
