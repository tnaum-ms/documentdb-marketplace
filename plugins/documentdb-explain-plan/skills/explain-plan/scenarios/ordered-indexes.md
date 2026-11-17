# Ordered vs Non-Ordered Index Scenarios

## When to Load This File
Load this scenario when:
- The user asks about ordered vs non-ordered indexes
- A COLLSCAN occurs despite a compound index existing (prefix rule violation)
- Sort pushdown is expected but not happening
- Index type needs to be determined

---

## Determining Index Type

You **cannot** determine whether indexes are ordered or non-ordered from the explain output alone. To find out:

**Option 1**: Ask the user to run `db.collection.getIndexes()` and look for `"storageEngine": {"enableOrderedIndexes": true}`.

**Option 2**: Ordered indexes are the default for current accounts; non-ordered indexes are only expected on very old accounts that have not been upgraded.

---

## Key Behavioral Differences

| Aspect | Non-Ordered (Legacy RUM) | Ordered (Composite) |
|--------|--------------------------|---------------------|
| Composite index storage | Separate tree per path | Single unified tree |
| Query on non-first path | ✅ Works (independent trees) | ❌ Requires first path filter |
| Range query performance | Poor (separate scans + intersect) | ✅ Single ordered scan |
| Sort pushdown | ❌ Not supported | ✅ Supported |
| $group pushdown | ❌ Not supported | ✅ Supported (if not multi-key) |
| Index-only scan | ❌ Not supported | ✅ Supported (if not truncated/multi-key) |
| Write performance | Worse (extra metadata terms) | ✅ Better (fewer terms) |
| Index size | Larger (exists terms, array terms) | ✅ Smaller |

---

## Ordered Index Prefix Rule

With ordered compound indexes `{a: 1, b: 1}`:
- ✅ `find({a: 5})` — uses index
- ✅ `find({a: 5, b: 10})` — uses index
- ❌ `find({b: 10})` — **cannot use index** (must filter on first path)

If you see a COLLSCAN where the user has a compound index but isn't filtering on the first field, this is likely the cause.

### Pattern 5: Compound Index Not Used (Prefix Rule)
**Symptom**: `COLLSCAN` despite having compound index `{a:1, b:1}` when query only filters on `b`
**Fix**: Create separate single-field index on `b`, or reorder compound index if `b` is always queried.

---

## Correlated Terms (Compound Index on Same-Array Paths)

Same-array compound paths can use two physical layouts:

- **Full-correlated legacy terms** store cross-element products. Separate
  dotted predicates may use multiple exact bounds because the cross-element
  combination exists in the index.
- **Reduced-correlated terms** store one tuple per array element.
  `hasCorrelatedTerms: true` in extended explain identifies this layout.
  Separate dotted predicates may match different elements, so unsafe secondary
  bounds are pruned and rechecked.

The layout is chosen when the index is **built**; use explain/index metadata
or build history to identify it.

For a reduced-correlated index, one `$elemMatch` owner can safely retain all
supported same-scope bounds when the service supports reduced-correlated
bounds planning and per-path metadata is available. With multiple `$elemMatch` owners, one owner
group drives a single scan; its same-scope fields may co-bound and the other
owners recheck. If that support or the metadata is unavailable, expect trim-to-leading
fallback. See `scenarios/multikey.md` for the complete decision model.

---

## Non-Ordered Index Preventing Sort Pushdown
**Symptom**: `SORT` stage even though an index exists on the sort field
**Fix**: Migrate to ordered index (`storageEngine: {enableOrderedIndexes: true}`)

---

## When to Recommend Migration to Ordered

Recommend migrating to ordered indexes when:
1. Sort pushdown is needed but blocked by non-ordered index
2. Compound index exists but uses bitmap internally (Pattern 9)
3. Range queries on compound indexes are slow (separate tree traversals)
4. Write performance or index size is a concern

**Migration command**:
```javascript
db.collection.createIndex(
  { field1: 1, field2: 1 },
  { storageEngine: { enableOrderedIndexes: true } }
)
```

**Caveat**: If existing queries rely on querying non-first paths of a compound index (which works with non-ordered but NOT with ordered), migration will break those queries. Always audit all queries using the index before recommending migration.
