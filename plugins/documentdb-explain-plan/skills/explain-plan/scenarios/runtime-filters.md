# Runtime Filter Scenarios

## When to Load This File
Load this scenario when:
- `totalDocsRemovedByRuntimeFilter` is high relative to `nReturned`
- `runtimeFilterSet` contains conditions that should ideally be in the index
- Good index scan selectivity but poor overall efficiency (many docs fetched then discarded)

---

## Index Filters vs Runtime Filters

- **`indexFilterSet`**: Conditions evaluated IN the index scan (efficient — reduces keys examined)
- **`runtimeFilterSet`**: Conditions evaluated AFTER fetching the document from heap (expensive if many docs pass the index but fail the runtime filter)

### The Problem
When `totalDocsRemovedByRuntimeFilter >> nReturned`, the index is letting through too many documents that ultimately don't match. The query does:
1. Index scan → finds N candidate docs
2. Fetch each doc from heap
3. Apply runtime filter → rejects most of them
4. Return the few that pass

Steps 2-3 are expensive per-document operations.

---

## Index Scan with High Runtime Filter Removal

**Symptom**: `IXSCAN` + `FETCH` with `totalDocsRemovedByRuntimeFilter` >> `nReturned`

**Efficiency Check**:
```
Fetch Ratio = totalDocsExamined / nReturned
```
- **≈ 1.0**: Good — almost every fetched doc is returned
- **>> 1.0**: ⚠️ Many docs fetched but filtered out by runtime filters

**Fix**:
- Add missing filter paths to a compound index
- Reorder compound index: put the most selective field first
- Check if negation operators are forcing runtime evaluation

---

## Why Conditions Become Runtime Filters

### 1. Missing from index
The most common case: the query filters on fields A, B, and C, but the index only covers A and B. Field C becomes a runtime filter.

### 2. Negation operators
`$ne`, `$nin`, `$not`, `$nor` — these cannot be efficiently pushed into index bounds on ordered indexes with arrays. They become runtime filters.

### 3. Correlated terms on compound indexes
First determine the index's physical layout:

- A **full-correlated** legacy index stores cross-element products, so separate
  same-array predicates may use multiple exact bounds.
- A **reduced-correlated** index (`hasCorrelatedTerms: true`) stores one tuple
  per array element. Separate predicates can match different elements, so
  unsafe secondary bounds become runtime filters.

For reduced-correlated indexes, one `$elemMatch` owner may retain all safe
same-scope bounds when the service supports reduced-correlated bounds
planning and per-path metadata is available. With multiple owners, one owner group drives
the scan and the others recheck. Otherwise the scan falls back to trimming
secondary same-array bounds. See `scenarios/multikey.md`.

### 4. Complex expressions
`$regex` (non-prefix), `$where`, `$expr` with complex logic — these cannot be expressed as index bounds.

### 5. $in with large arrays on non-leading paths
On ordered indexes, `$in` on non-leading paths may not generate tight bounds, causing broader scans with runtime filtering.

---

## $in with Large Array + Range Filter

**Symptom**: Multiple scan boundaries, high `totalKeysExamined`, runtime filters active
**Fix**:
- With ordered indexes, `$in` generates `(value, range_start) → (value, range_end)` per $in value
- Create a compound index that places the $in field first for tighter bounds
- If the $in array is very large (100+), consider restructuring the query

---

## Recommendations

### Primary Fix: Compound Index
Create a compound ordered index that includes ALL filter fields:
```javascript
// Query: find({status: "active", region: "us-east", priority: {$gte: 5}})
// Current index: {status: 1} — region and priority are runtime filters

// Better:
db.collection.createIndex(
  { status: 1, region: 1, priority: 1 },
  { storageEngine: { enableOrderedIndexes: true } }
)
```

**Field ordering in compound index**:
1. Equality fields first (most selective equality field leading)
2. Sort or group-key fields next (if index ordering is needed)
3. Range fields last

This follows ESR (Equality → Sort → Range). Putting a range before the
sort/group fields destroys their global index order and forces an in-memory
sort.

### When You Can't Eliminate Runtime Filters
- Negation operators: Accept the runtime filter or restructure query to avoid `$ne`/`$nin`
- Correlated terms: Determine full- versus reduced-correlated layout first;
  for reduced terms, check per-path metadata and the selected `$elemMatch`
  owner before deciding which paths must recheck
- Complex expressions: These can't be indexed; minimize the docs reaching this stage by ensuring earlier index conditions are selective
