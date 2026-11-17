# COLLSCAN Scenarios

## When to Load This File
Load this scenario when:
- `stage: "COLLSCAN"` appears in the explain plan
- No index is being used despite filter conditions existing
- High `totalDocsExamined` with no index scan

---

## What COLLSCAN Means

A COLLSCAN (collection scan) means the query engine is reading every document in the collection sequentially. This is:
- **O(n)** where n = total documents in collection
- Always bad for large collections (>10K docs)
- Acceptable for tiny collections or intentional full-table operations

---

## COLLSCAN on Large Collection

**Symptom**: `stage: "COLLSCAN"`, high `totalDocsExamined`
**Fix**: Create an index on the filter fields. For ordered indexes, ensure the first path matches the most common equality filter.

### Choosing the Right Index

1. **Single equality filter**: `db.coll.find({status: "active"})` → `createIndex({status: 1})`
2. **Multiple equality filters**: `db.coll.find({status: "active", type: "user"})` → `createIndex({status: 1, type: 1})` — put most selective first
3. **Equality + range**: `db.coll.find({status: "active", age: {$gt: 18}})` → `createIndex({status: 1, age: 1})` — equality before range
4. **Equality + sort**: `db.coll.find({status: "active"}).sort({createdAt: -1})` → `createIndex({status: 1, createdAt: -1})`

---

## Compound Index Not Used (Prefix Rule)

**Symptom**: `COLLSCAN` despite having compound index `{a:1, b:1}` when query only filters on `b`

**Root Cause**: Ordered compound indexes require the first path to be in the filter. If your query only uses `b`, the index `{a:1, b:1}` cannot be used.

**Fix**:
- Create a separate single-field index on `b`
- OR reorder the compound index to `{b:1, a:1}` if `b` is always queried

---

## Full Scan Type on Index (Effective COLLSCAN)

Sometimes the explain shows an IXSCAN but with `scanType: "full"`, meaning the entire posting tree is scanned. This is effectively a COLLSCAN through the index.

**Symptom**: `scanType: "full"` in indexUsage
**Causes**:
- `$exists: true` on a high-cardinality field (matches everything)
- Negation operators like `$ne` or `$nin` (can't narrow bounds)
- Missing or overly broad filter on the indexed path

**Fix**: Add a more selective leading field to a compound index.

---

## COLLSCAN Diagnosis Questions

1. **Does any index exist for the filter fields?** → If no, create one
2. **Is there a compound index but query skips the first path?** → Prefix rule violation (Pattern 5)
3. **Is the filter using negation ($ne, $nin, $not)?** → These often can't use indexes efficiently
4. **Is the collection very small (<1K docs)?** → COLLSCAN may be optimal (planner chose it intentionally)
5. **Is a hint forcing a different index?** → The planner may have rejected the relevant index
