# Bitmap AND/OR Scenarios

## When to Load This File
Load this scenario when the explain plan contains:
- `stage: "AND"` or `stage: "OR"` (bitmap intersection/union)
- Multiple `IXSCAN` inputs under a single parent stage
- `isBitmap: true` on IXSCAN stages with high key counts

---

## How Bitmap Scans Work in DocumentDB

Bitmap scans collect TID sets from index scans, then combine them:
- **Bitmap AND**: Intersects TID sets — only docs matching ALL conditions
- **Bitmap OR**: Unions TID sets — docs matching ANY condition (used for `$or`)

After combining, a single Bitmap Heap Scan fetches documents by page.

### Key Fields
- `isBitmap: true`: Confirms bitmap mode
- `exactBlocksRead` / `lossyBlocksRead`: Page-level behavior (lossy = page rechecks needed)
- `totalDocsRemovedByIndexRechecks`: Docs rejected during bitmap recheck (normal overhead)
- `numBlocksFromCache` / `numBlocksFromDisk`: Cache efficiency

---

## Multi-Index Join with Asymmetric Selectivity (Most Common!)

**Symptom**: `AND` or `OR` stage (bitmap) with multiple `IXSCAN` inputs where one scans millions of keys and another scans very few. Total execution time dominated by the unselective scan.

**Root Cause**: The PostgreSQL planner lacks accurate selectivity statistics and chooses to bitmap-intersect/union multiple single-field indexes instead of using the most selective one alone (or a compound index). This affects any multi-index join pattern — bitmap AND for `$and` filters, bitmap OR for `$or` queries.

**Detection Heuristic**:
```
If IXSCAN_A.totalKeysExamined >> IXSCAN_B.totalKeysExamined (e.g., 10x+)
AND the AND/OR parent nReturned is close to the smaller scan
→ The large scan is wasted work
```

**Fix**:
- **Primary**: Create a compound ordered index with the selective equality field first:
  ```javascript
  db.collection.createIndex(
    { "selectiveField": 1, "broadField": 1 },
    { storageEngine: { enableOrderedIndexes: true } }
  )
  ```
- **Alternative**: Reach out to the Query & Indexing team about enabling **custom statistics** for the planner. This feature collects PostgreSQL MCV histograms that allow the planner to correctly estimate selectivity of each filter/index and choose an index scan on the most selective index, pushing the other condition as a cheap runtime filter. Combined with the **composite index planner**, this is the best long-term route for multi-index join problems.

---

## Non-Ordered Compound Index with Slow Bitmap Scan

**Symptom**: `isBitmap: true` on a compound index, perfect selectivity (keysExamined ≈ nReturned ≈ docsExamined) but disproportionately slow execution. High `numBlocksFromCache` at IXSCAN relative to result count.

**Root Cause**: Non-ordered compound indexes store each path as a separate posting tree. Even though the compound index exists, it does separate tree traversals + bitmap intersection instead of a single ordered walk. The posting tree traversal is the bottleneck.

**Detection**:
- Good selectivity ratios (efficiency ≈ 1.0) but slow time
- `isBitmap: true` even on a compound index
- Index is non-ordered (no `enableOrderedIndexes: true`)

**Fix**: Migrate to an ordered index with the same key pattern. An ordered compound index stores all paths in a single B-tree, enabling direct range scans.

---

## Bitmap OR for $or Queries

When a query uses `$or`, the planner may choose bitmap OR across multiple indexes:
```javascript
db.coll.find({ $or: [{ status: "active" }, { priority: "high" }] })
// → OR stage with two IXSCAN children (one per index)
```

This is **acceptable** when both branches are similarly selective. It becomes a problem when:
- One branch is highly selective (few matches) and the other is broad
- The broad branch dominates execution time

**Fix for asymmetric $or**:
- If possible, restructure the query to separate the selective condition
- Create a compound index that covers the common case
- Consider whether a `$unionWith` or two separate queries merged client-side would be faster

---

## Analysis Questions for Bitmap Plans

When you encounter a bitmap plan, ask:
1. **Are the index scans similarly selective?** Compare `totalKeysExamined` across children.
2. **Is the bitmap AND/OR the bottleneck?** Check if the bitmap stage's time dominates total.
3. **Would a single compound index eliminate the multi-index join?** The compound index must have the most selective equality field first.
4. **Is the index ordered or non-ordered?** Non-ordered compounds always use bitmap internally.
