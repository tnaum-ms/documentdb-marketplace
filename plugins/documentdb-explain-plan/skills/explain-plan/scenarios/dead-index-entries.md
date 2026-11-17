# Dead Index Entry Scenarios

## When to Load This File
Load this scenario when DocumentDB explain output includes:
- `IXSCAN` under `LIMIT`, returning very few documents (`nReturned` is small)
- `executionStats.executionStages.indexUsage[].scanLoops` much larger than `nReturned`
- `deadEntriesOrPagesSkipped` present or non-zero
- `totalKeysExamined` much larger than `nReturned`, especially when the `bounds` are broad and every eligible live entry in the range should satisfy the predicate
- High `numBlocksFromDisk`, `exactBlocksRead`, or low cache hit ratio while returning very few rows
- High `executionStartAtTimeMillis` that is close to `executionTimeMillis`, meaning most time was spent before the first row was produced
- `indexUsage.bounds` like `MinKey`/`MinDate` to a cutoff for `$lte`, or a cutoff to `MaxKey`/`MaxDate` for `$gte`
- Time-series or log collections where TTL/delete churn is plausible

If the user provides raw PostgreSQL EXPLAIN instead of DocumentDB JSON, translate it into these DocumentDB concepts: `innerScanLoops` maps to `scanLoops`, and `actual time=start..end` maps to `executionStartAtTimeMillis` vs `executionTimeMillis`.

---

## Core Diagnostic

Do **not** automatically diagnose high keys examined, high block reads, or slow execution as a bad query direction, poor selectivity, or cold cache. First ask whether the index entries in the DocumentDB `bounds` should have matched the query if they were live.

Example pattern (executionStats only — the key signals):
```json
"executionStats": {
  "nReturned": 4,
  "executionStartAtTimeMillis": 31480,
  "executionTimeMillis": 31486,
  "totalKeysExamined": 51207,
  "executionStages": {
    "stage": "LIMIT",
    "inputStage": {
      "stage": "IXSCAN",
      "nReturned": 4,
      "totalKeysExamined": 51207,
      "numBlocksFromDisk": 51210,
      "indexUsage": [{
        "scanType": "ordered",
        "scanLoops": 51207,
        "deadEntriesOrPagesSkipped": 51203
      }]
    }
  }
}
```
Key signal: bounds are `[MinDate, 2026-01-15]` (broad `$lte`), so every live entry should match — yet `scanLoops` (51K) >> `nReturned` (4).

For a `$lte` bound from `MinDate` to a cutoff, every eligible live index entry in that range should satisfy the predicate. If `totalKeysExamined` is 51,207 but `nReturned` is 4, the likely explanation is not "51,203 non-matching rows"; it is that many scanned index entries are dead or not visible to the current snapshot and had to be checked/skipped before 4 visible rows were found.

Also inspect `executionStartAtTimeMillis`. When it is very high and close to `executionTimeMillis`, the scan did almost all of its work before producing the first row. With `LIMIT 4`, that is a strong clue that the runtime had to walk and reject/skip many entries before finding any visible row, which is consistent with dead or invisible index entries.

---

## Common Root Causes

1. **TTL deletes or heavy delete churn**
   - Time-series/log collections often use TTL indexes or periodic deletes.
   - The logical documents may be gone, but dead index tuples can remain until vacuum/pruning catches up.

2. **Vacuum lag**
   - Autovacuum may not have run recently or may not be keeping up with churn.
   - Recent vacuum may still be unable to clean if replicas lag or long-running snapshots hold old row versions.

3. **Lagging replicas or long-running queries**
   - These can prevent dead tuples from being pruned even after deletes occur.

4. **LP_DEAD opportunity**
   - If the runtime repeatedly encounters dead index entries, LP_DEAD-style marking/pruning can avoid returning those entries to the runtime on future scans.

---

## What to Check

Ask for or inspect:
- Whether the collection has a TTL index or regular delete workload.
- Long-running queries or open snapshots during the slow period.
- Whether repeated executions improve after cache warmup or over time as cleanup runs. If the problem goes away after cleanup, dead entries were likely the root cause.

---

## How to Frame the Finding

Use wording like:

> The ordered scan walked 51K index entries but returned only 4 rows. Because the bound starts at MinDate, those entries should have matched the predicate if they were live. That points to dead or invisible index entries from deletes/TTL churn, not just a broad range scan. Check TTL/delete activity and long-running queries.

If the timing supports it, add:

> The time to first row was almost the full execution time, so the bottleneck happened before the scan produced even the first visible result. That reinforces the dead/invisible-entry hypothesis.

Avoid overconfident recommendations such as:
- "Add `.sort({createdAt: -1})` and it will read only 4 entries"
- "Create a descending index"
- "Increase shared buffers"

Those may still be useful secondary ideas for a query that intentionally wants newest rows or suffers cold-cache IO, but they do not explain why a broad matching range required tens of thousands of loops before finding 4 visible rows.

---

## Recommended Actions

1. **Validate delete/TTL churn**
   - Check whether the collection has TTL indexes or scheduled deletes.
   - Correlate delete volume with the slow query window.

2. **Check blockers**
   - Look for long-running queries or snapshots that prevent pruning.

3. **If it persists**
   - If dead entries keep dominating scan loops, open an Azure support request and include this explain output.
