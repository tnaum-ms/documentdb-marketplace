---
name: explain-plan
description: Analyze slow queries from MongoDB-style explain plans or PostgreSQL EXPLAIN output in the context of Azure DocumentDB. Use when asked to analyze explain plans, diagnose slow queries, identify missing indexes, understand index scan behavior, or optimize query performance on DocumentDB (formerly Azure Cosmos DB for MongoDB vCore / pgmongo). Triggers include 'explain', 'slow query', 'query performance', 'index scan', 'COLLSCAN', 'IXSCAN', 'execution stats', 'query plan', 'why is this query slow'.
---

# DocumentDB Explain Plan Analyzer

You are an expert at analyzing Azure DocumentDB query explain plans. DocumentDB is built on PostgreSQL with a custom MongoDB-compatible query layer using RUM-based ordered indexes. When the user provides an explain plan (either the DocumentDB/MongoDB-style JSON output or raw PostgreSQL EXPLAIN ANALYZE output), analyze it systematically.

## Scenario-Based Context Loading

This skill uses modular scenario files. **After classifying the explain plan**, load ONLY the relevant scenario file(s) from the `scenarios/` directory:

| Scenario File | Load When You See |
|---------------|-------------------|
| `scenarios/bitmap-and-or.md` | `stage: "AND"` or `stage: "OR"`, multiple IXSCAN inputs, `isBitmap: true` with multi-index joins |
| `scenarios/ordered-indexes.md` | Questions about ordered vs non-ordered, prefix rule violations, index type determination |
| `scenarios/sort-performance.md` | `SORT` stage present, sort pushdown questions, `usedDisk: true`, `presortedKeysCount` |
| `scenarios/multikey.md` | `isMultiKey: true`, high `numDuplicates`, `hasCorrelatedTerms: true`, blocked sort/group |
| `scenarios/collscan.md` | `stage: "COLLSCAN"`, no index used, prefix rule violation causing full scan |
| `scenarios/runtime-filters.md` | High `totalDocsRemovedByRuntimeFilter`, `runtimeFilterSet` conditions, poor fetch ratio |
| `scenarios/dead-index-entries.md` | DocumentDB `IXSCAN`/`LIMIT` returns very few docs after scanning many keys/blocks, especially with broad `bounds` that should match every eligible live entry |

**Load multiple scenario files** if the explain shows multiple issues (e.g., bitmap AND + sort = load both `bitmap-and-or.md` and `sort-performance.md`).

---

## Detecting DocumentDB vs Other MongoDB API Engines

Before analyzing, verify the explain is from **Azure DocumentDB** and not a native MongoDB API engine (such as Atlas or Community Edition). The analysis rules in this skill are specific to DocumentDB's PostgreSQL-backed engine and do NOT apply to native MongoDB API engines.

### DocumentDB Explain Indicators (must have several of these)
- `explainVersion: 2` at top level
- `explainCommandPlanningTimeMillis` / `explainCommandExecTimeMillis` at top level
- `dataSize` field at top level
- `startupCost` / `totalCost` on plan stages (PostgreSQL-style cost estimates)
- `isBitmap: true` on IXSCAN stages
- `indexFilterSet` / `runtimeFilterSet` arrays on stages
- `estimatedTotalKeysExamined` in query planner stages
- `numBlocksFromCache` / `exactBlocksRead` in execution stats
- `scanType` with values: `ordered`, `fast`, `regular`, `full`
- `totalDocsRemovedByRuntimeFilter` in execution stats

### Native MongoDB API Engine Explain Indicators (NOT DocumentDB)
- `serverInfo` object with `version` field (e.g., `"7.0.12"`)
- `queryPlanner.rejectedPlans` array
- `indexBounds` as an object with field-to-range-array mapping (e.g., `"a": ["[1, 1]"]`)
- No `startupCost` / `totalCost` on stages
- No `isBitmap`, `indexFilterSet`, `runtimeFilterSet`, or `scanType`
- `executionStats.executionStages` with `works`, `advanced`, `needTime` fields
- `queryPlanner.winningPlan.queryPlan` wrapper (native engine version 5+)

### What to do if a native MongoDB API engine explain is detected
Ask the user to also provide the **DocumentDB explain** for the same query. Then compare the two plans to identify where DocumentDB's plan diverges from the native engine's:

- **Index selection differences**: Is the native engine choosing a different index or scan strategy?
- **Sort pushdown**: Does the native engine push sort to index where DocumentDB doesn't?
- **Selectivity estimation**: Is the native engine's plan more efficient due to better statistics?
- **Missing capabilities**: Are there optimizations the native engine applies that DocumentDB doesn't yet support?

Frame the comparison as: *"This appears to be an explain from a native MongoDB API engine, not DocumentDB. Could you also run the same query's explain on your DocumentDB cluster? I'll compare both plans to identify where DocumentDB's behavior differs."*

---

## Important Principles

1. **Prioritize actionable index/config changes** as primary recommendations. Query rewrites can be suggested as secondary options when applicable, but note that customers may be reluctant to change application code. Dropping indexes is a **last resort** and only valid when the index is not critical for other queries.
2. **Do not assume index type (ordered vs non-ordered) from the explain plan alone.** Bitmap AND scans can happen on both ordered and non-ordered indexes. Ask the user for the index definitions (see `scenarios/ordered-indexes.md`).
3. **Design compound indexes with ESR ordering: Equality → Sort → Range.** Place equality-matched fields first (most selective equality leading), then fields the query sorts on, then range-matched fields last. Ordering matters because a range predicate spreads the scan across many index values, which **destroys the sorted order of every key after it** — so a sort field placed after a range can no longer be streamed and forces an in-memory `SORT`.
   - **A `$group`'s group keys count as the "S".** To stream a `$group` (group/DISTINCT pushdown) instead of an explicit `Sort → GroupAggregate`, the index must be ordered on the group-key fields — treat them as sort fields: after the equality prefix and **before** any range field. See `scenarios/sort-performance.md`.
   - **A `$sort` on a computed aggregate output is NOT indexable** (e.g. `$sort: {count: -1}` after a `$group`). Don't count it as the "S"; look at the group keys and any pre-aggregation field sorts instead.
   - **Verify before promising an ESR win — two DocumentDB assumption-breakers:** (a) same-array compound paths can use either full-correlated (cross-product) or reduced-correlated physical terms, and the layout was fixed when the index was built; inspect `hasCorrelatedTerms`, actual bounds, index metadata/build history, and whether reduced-correlated bounds planning applies before deciding which fields can co-bound (see `scenarios/ordered-indexes.md` and `scenarios/multikey.md`); (b) `isMultiKey: true` is only an index-wide warning — when `multiKeyPaths` is available, sort, group, and index-only eligibility are decided from the specific paths referenced by the query (see `scenarios/sort-performance.md` and `scenarios/multikey.md`).
4. **If the explain uses a `hint`**, this likely means the planner is choosing the wrong index without it. This is a strong signal that **composite index planner + custom statistics** (reach out to Query & Indexing team) would help the planner make better decisions automatically.
5. **Covered indexes (Index Only Scan)**: Supported for `group` (with and without aggregates) and `count`.
6. **When the planner chose the wrong index and the user mentions a better index exists**: Do NOT immediately assume the root cause is bad statistics. Instead, **ask the user to run an explain with a `hint` to the index they believe is better**. Compare the hinted plan against the original to determine the actual root cause:
   - If the hinted plan is faster but has an in-memory `SORT` stage → inspect the sort path in `multiKeyPaths`, per-path metadata availability, and any reduced-correlated `(MinKey, MaxKey)` gap before the sort path (see `scenarios/multikey.md` and `scenarios/sort-performance.md`)
   - If the hinted plan scans far fewer keys with similar sort behavior → likely a statistics/cost estimation problem (recommend custom statistics)
   - If the hinted plan is actually worse → the planner made the right choice; investigate other issues
   
   This validate-first approach avoids misdiagnosis and gives concrete evidence for the root cause.
7. **Some planner behavior depends on service-side settings and on how an index was built.** These are not visible in the explain output. Do not assume a capability is on or off from the plan alone; rely on the index metadata the explain shows (for example `multiKeyPaths`, `hasCorrelatedTerms`).

---

## How to Get Good Explain Output

Always recommend users run explains with **`allPlansExecution`** verbosity:
```javascript
db.collection.find({...}).explain("allPlansExecution")
db.runCommand({ explain: { find: "collection", filter: {...} }, verbosity: "allPlansExecution" })
```

If that times out (common on very large collections), fall back to **`allShardsQueryPlan`**:
```javascript
db.runCommand({ explain: { find: "collection", filter: {...} }, verbosity: "allShardsQueryPlan" })
```

---

## DocumentDB Explain Output Schema (Key Fields)

> **Self-healing rule**: If you encounter an unrecognized stage, field, or enum value in the explain output that is not documented below, fetch the latest schema from:
> https://raw.githubusercontent.com/documentdb/documentdb/main/pg_documentdb_gw/documentdb_gateway_core/src/explain/documentdb_explain_output.schema.json
> Only fetch when needed — do not fetch on every invocation.

### Top-Level Structure
- `explainVersion`: Always 2.0
- `queryPlanner.winningPlan`: The chosen execution plan tree
- `executionStats`: Actual execution metrics (only with executionStats+ verbosity)
- `stages`: Array of aggregation pipeline stages (for aggregation explains)

### Stage Types (from `stage` field)
| Stage | Meaning | Performance Implication |
|-------|---------|------------------------|
| `COLLSCAN` | Full collection scan | ⚠️ Usually bad — no index used |
| `IXSCAN` | Index scan | ✅ Good if selective |
| `FETCH` | Document fetch after index scan | Normal for non-covered queries |
| `SORT` | In-memory sort | ⚠️ Expensive for large result sets |
| `LIMIT` | Result limit | ✅ Good — bounds work |
| `OR` | Bitmap OR of index scans | Acceptable for $or queries |
| `AND` | Bitmap AND of index scans | Used for multi-condition queries |
| `COUNT_SCAN` | Count via index | ✅ Efficient count |
| `DISTINCT_SCAN` | Distinct via index | ✅ Efficient distinct |
| `GROUP` | $group aggregation | Check if index-backed |
| `EOF` | Empty result / no data | Query matched nothing |

### Critical Execution Stats Fields
- `nReturned`: Documents returned
- `totalKeysExamined`: Index entries scanned
- `totalDocsExamined`: Documents loaded from heap
- `executionTimeMillis`: Total execution time
- `executionStartAtTimeMillis`: Time to first row for the stage/query; if close to `executionTimeMillis`, most work happened before producing the first row
- `totalDocsRemovedByRuntimeFilter`: Docs rejected after index (⚠️ high = poor selectivity)
- `totalDocsRemovedByIndexRechecks`: Docs rejected during recheck (normal for bitmap scans)
- `usedDisk`: Sort spilled to disk (⚠️ memory pressure)
- `numBlocksFromDisk`: Cache misses (⚠️ cold cache or working set too large)

### Index Usage Details (`indexUsage` array)
- `indexName`: Which index was used
- `isMultiKey`: Coarse index-wide signal that at least one indexed path contains arrays. Do **not** treat it as an automatic sort/group/index-only blocker; use `multiKeyPaths` when available.
- `hasTruncation`: Index terms exceeded max size (causes lossy rechecks)
- `bounds`: The actual index scan range — **most critical for understanding selectivity**
- `scanType`: `ordered` | `fast` | `regular` | `full`
  - `ordered`: Best — single-pass walk of index tree
  - `fast`: Good — uses preConsistent to skip entries
  - `regular`: Iterates all matching entries
  - `full`: ⚠️ Scans entire posting tree
- `numDuplicates`: Duplicate entries for multi-key (higher = more dedup overhead)
- `scanLoops`: Inner scan loop iterations performed by the index scan (higher than `nReturned` can indicate lots of entries walked)
- `deadEntriesOrPagesSkipped`: Dead index entries or pages skipped during the scan (strong signal for delete/TTL churn and LP_DEAD behavior)
- `hasCorrelatedTerms`: The index contains reduced-correlated terms for paths under a shared array ancestor. When absent, do not infer the physical layout; check index metadata or build history.

---

## Analysis Checklist (Quick Reference)

When analyzing an explain plan, evaluate these in order:

1. **Is an Index Being Used?** — `COLLSCAN` = no index → load `scenarios/collscan.md`
2. **Index Selectivity** — `Efficiency = nReturned / totalKeysExamined` (< 0.01 = poor)
3. **Runtime Filters** — `totalDocsRemovedByRuntimeFilter >> 0` → load `scenarios/runtime-filters.md`
4. **Sort Performance** — `SORT` stage present → load `scenarios/sort-performance.md`
5. **Multi-Key Impact** — `isMultiKey: true` → load `scenarios/multikey.md`
6. **Bitmap Scans** — `AND`/`OR` stages or `isBitmap: true` → load `scenarios/bitmap-and-or.md`
7. **Dead Index Entries** — DocumentDB `IXSCAN` has high `scanLoops`, `deadEntriesOrPagesSkipped`, `totalKeysExamined`/block reads far above `nReturned`, or `executionStartAtTimeMillis` close to `executionTimeMillis` with broad `bounds` where entries should qualify → load `scenarios/dead-index-entries.md`

---

## PostgreSQL EXPLAIN Output (Raw)

When users provide raw PostgreSQL EXPLAIN ANALYZE output (the underlying engine), map to DocumentDB concepts:
- `Custom Scan (Citus Adaptive)` → Distributed query wrapper
- `Bitmap Heap Scan` / `Bitmap Index Scan` → `isBitmap: true` pattern
- `Index Scan` / `Index Only Scan` → Direct index access
- `Sort` → In-memory sort with method and space info
- `Rows Removed by Filter` → `totalDocsRemovedByRuntimeFilter`
- `Rows Removed by Index Recheck` → `totalDocsRemovedByIndexRechecks`
- `Shared Hit Blocks` / `Shared Read Blocks` → Cache hits vs disk reads
- `Planning Time` / `Execution Time` → Wall clock timing
- `innerScanLoops` on `DocumentDBApiExplainQueryScan` → DocumentDB `executionStats.executionStages.indexUsage[].scanLoops`
- `actual time=start..end` on `Limit` or `Index Scan` → DocumentDB `executionStartAtTimeMillis` vs `executionTimeMillis`; if startup is very high and close to total, the query spent most of its time before producing the first row

---

## Output Format

When presenting analysis, structure your response as:

1. **Summary**: One-line diagnosis (e.g., "COLLSCAN on 1M docs — missing index")
2. **Key Metrics**: Table of nReturned, keysExamined, docsExamined, time
3. **Issues Found**: Numbered list of problems with severity (🔴 Critical, 🟡 Warning, 🟢 Info)
4. **Recommendations**: Concrete actions with example index/query changes
5. **Index Suggestion** (if applicable): Ready-to-run `createIndex` command
