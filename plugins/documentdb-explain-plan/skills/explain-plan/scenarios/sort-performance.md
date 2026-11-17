# Sort Performance Scenarios

## When to Load This File
Load this scenario when:
- A `SORT` stage appears in the explain plan
- The user asks why sort isn't pushed to the index
- `usedDisk: true` or large `totalDataSizeSortedBytesEstimate`
- `presortedKeysCount` indicates incremental sort

---

## Sort Pushdown Rules (Ordered Indexes Only)

Sort can be pushed to the index (avoiding in-memory SORT) when ALL of these hold:

1. **Index is ordered**: The index must be an ordered index (`orderedIndex: true` in index metadata). Legacy non-ordered RUM indexes cannot support sort pushdown.
2. **Sort path matches an index key**
3. **Equality prefixes before the sort path**: e.g., `index: {a:1, b:1}` with `find({a:5}).sort({b:1})` ✅
4. **Direction match**: Sort direction matches index direction, OR complete reverse (backward scan)

### Multi-value `$in` on an equality prefix requires merge-sort support

Do not treat a multi-value `$in` prefix as equivalent to one fixed equality for ordering. For an ordered index such as `{tenantId: 1, itemId: 1, eventTime: -1, createdAt: -1}`, each distinct `tenantId` value produces a separately ordered stream. The index alone does not provide global `{itemId, eventTime, createdAt}` order across those streams.

To push the trailing sort and stream a following `$group`, DocumentDB must merge the per-prefix streams. Whether the service supports this merge is not visible in the explain output; validate it with a hinted explain.

When this pattern appears:
- Deduplicate the `$in` list before testing if it contains duplicate values.
- Design the ordered index as equality/`$in` prefix → sort/group keys → range keys.
- Run a hinted `EXPLAIN ANALYZE` against the candidate index.
- Successful pushdown should remove the explicit pre-group `Sort` nodes; a merge node may remain to preserve global order across streams or workers.
- If the merge is unsupported, expect an explicit sort before `GroupAggregate` even when the index keys otherwise match.
- Do not confuse this pre-group sort with a final post-group sort on different or computed output fields; the latter may remain non-indexable.

---

## $group Ordering Is a "Sort" for Index Design (ESR)

A `$group` doesn't just aggregate — it imposes an **ordering requirement on the group-key fields** if you want it to stream from the index (group/DISTINCT pushdown) instead of doing an explicit `Sort → GroupAggregate`. When designing or evaluating an index for an aggregation, treat the **group keys as the "S" in ESR** (Equality → Sort → Range):

- Group keys belong **after** the equality prefix and **before** any range field. A range placed before the group keys spreads the scan across many values and destroys group-key order → forces an explicit sort to feed the aggregate.
- Example: for `$match: {a: "x", ts: {$gte..$lte}}` then `$group: {_id: {t: "$type", s: "$subType"}}`, the ESR-honest index is `{a, type, subType, ts}` (E, S, S, R) — **not** `{a, ts, type, subType}` (E, R, S, S), which cannot stream the group.

### Don't count a computed-output `$sort` as the "S"
A `$sort` on a value produced *by* the pipeline (e.g., `$sort: {count: -1}` after a `$group`) sorts on data that exists only after aggregation — **no index can serve it.** When two sorts appear in a plan (e.g., `Sort → GroupAggregate` on the group key, then a final `Sort` on `count`), the **indexable** one is the group-key sort. Design against that; ignore the computed sort.

### Caveat: multikey group pushdown is gated per path

Where the service does not support multi-key group pushdown, an
`isMultiKey: true` index conservatively blocks streaming group order. Where it
does and `multiKeyPaths` is available, a multi-key equality prefix can still feed a
streaming group if the group/order columns are scalar. A group/order path that
is itself multi-key remains blocked. Reduced-correlated prefix bounds and the
ordinary ESR rules must also be sound.

---

## Additional Multi-Key Constraints Are Per Path

Start with `multiKeyPaths`, not the index-wide `isMultiKey` flag. If
`multiKeyPaths` is absent, the planner cannot prove which path is scalar and
falls back conservatively.

### Summary Table

| Condition | Sort path proven scalar | Sort path multi-key or unknown |
|-----------|-------------------------|--------------------------------|
| AM supports ordering | Required ✅ | Required ✅ |
| Sort path matches index key | Required ✅ | Required ✅ |
| Equality prefixes before sort path | Required ✅ | Required ✅ |
| No filter on sort path | Allowed ✅ | Allowed ✅ |
| Full-domain filter (`$exists: true`, `MinKey → MaxKey`) | Allowed ✅ | Allowed ✅ |
| Narrowing equality/range on sort path | Allowed ✅ | **Blocks pushdown** ❌ |
| Reverse direction | Backward scan ✅ | Backward scan ✅ |

The per-path decision is honored only when the index carries per-path
metadata (`multiKeyPaths`).

### Concrete Examples

```javascript
// Index {a: 1, b: 1}; a is multi-key, b is scalar according to multiKeyPaths.

// ✅ Scalar sort path with equality prefix.
db.coll.find({a: 5}).sort({b: 1})

// ✅ Scalar sort path may also carry a range.
db.coll.find({a: 5, b: {$gt: 3}}).sort({b: 1})

// ✅ Multi-key sort path with no narrowing filter.
db.coll.find().sort({a: 1})

// ✅ $exists:true lowers to a full-domain MinKey→MaxKey bound, so ordering survives.
db.coll.find({a: {$exists: true}}).sort({a: 1})

// ❌ Multi-key sort path with a narrowing range.
db.coll.find({a: {$gt: 3}}).sort({a: 1})

// ❌ Multi-key sort path with a narrowing equality.
db.coll.find({a: 5}).sort({a: 1})
```

### `$group` Example

For index `{labels: 1, category: 1}` where `labels` is multi-key and
`category` is scalar:

- Where multi-key group pushdown is supported and per-path metadata exists,
  `match({labels:"x"}) → group({_id:"$category"})` may stream as a
  `GroupAggregate`.
- Grouping by `$labels` remains blocked because the grouped/ordered path itself
  is multi-key.

---

## Incremental Sort

When the index provides partial ordering, PostgreSQL may use incremental sort:
- `presortedKeysCount > 0` in explain means some keys come pre-sorted from index
- Remaining keys are sorted in-memory in smaller batches
- Better than full sort but worse than full pushdown

---

## In-Memory Sort on Large Result Set

**Symptom**: `SORT` stage with `usedDisk: true` or large `totalDataSizeSortedBytesEstimate`
**Fix**:
- Create/modify index to match sort order
- Add `$limit` before sort if possible
- Check sort direction matches index direction
- If the index is multi-key, inspect `multiKeyPaths` instead of rejecting the
  whole index

---

## Non-Ordered Index Preventing Sort Pushdown

**Symptom**: `SORT` stage even though an index exists on the sort field
**Fix**: Migrate to ordered index (`storageEngine: {enableOrderedIndexes: true}`)

---

## Multi-Key Index Blocking Sort/Group Pushdown

**Symptom**: `isMultiKey: true`, `SORT`/`HashAggregate` appears despite an
apparently matching index.

**Fix**:
- Check whether the actual sort/group path is listed in `multiKeyPaths`
- If `multiKeyPaths` is absent, treat the index as lacking usable per-path metadata
- Distinguish a narrowing filter from a full-domain `$exists: true` bound
- For reduced-correlated prefixes, confirm safe bounds were retained; a
  `(MinKey, MaxKey)` gap before the sort path breaks index ordering
- Then check ordinary equality-prefix and direction rules

---

## Diagnosis Flowchart

When you see a SORT stage:
1. **Is the index ordered?** → If no, recommend migration (Pattern 4)
2. **Is there an index on the sort path?** → If no, recommend creating one
3. **Are equality prefixes satisfied?** → If no, reorder index or add missing prefix filter
4. **Does an equality prefix use multi-value `$in`?** → If yes, deduplicate it and validate with a hinted explain
5. **Is the index multi-key?** → Inspect `multiKeyPaths` and metadata availability;
   do not stop at `isMultiKey: true`
6. **Does sort direction match?** → If reversed, check if backward scan is possible
7. **Is it incremental sort?** → Partial win; check if full pushdown is achievable
