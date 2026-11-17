# Multi-Key Index Scenarios

## When to Load This File
Load this scenario when:
- `isMultiKey: true` appears in the explain output
- `numDuplicates` is high
- Sort or $group pushdown is unexpectedly blocked
- `hasCorrelatedTerms: true` appears

---

## What Makes an Index Multi-Key

An index becomes multi-key when the indexed field contains an array in at least one document. This fundamentally changes the index behavior because:
- Each array element generates a separate index entry for the same document
- The document appears multiple times in the index (once per array element)
- Ordering guarantees break down (document appears at multiple positions)

---

## Impact on Query Operations

### Sort Pushdown Is Decided Per Path

`isMultiKey: true` means **some** index path contains arrays; it does not mean
every path is multi-key. When `multiKeyPaths` is present, use it to classify the
actual sort path:

| Sort-path state | Filter on sort path | Pushdown |
|-----------------|---------------------|----------|
| Proven scalar (not in `multiKeyPaths`) | Equality or range | Allowed if the normal ordered-index prefix/direction rules hold ✅ |
| Multi-key | No filter | Allowed ✅ |
| Multi-key | Full-domain bound such as `$exists: true` (`MinKey → MaxKey`) | Allowed because the bound does not narrow the ordering domain ✅ |
| Multi-key | Narrowing equality or range | Blocked ❌ |
| Unknown (no usable per-path metadata) | Narrowing equality or range | Conservatively blocked ❌ |

Per-path sort pushdown only helps indexes that actually carry per-path
metadata; use `multiKeyPaths`/index metadata to determine whether the existing
index was built with usable tracking.

### `$group` Ordering Is Configuration- and Path-Dependent

Where the service does not support multi-key group pushdown, a group-by over
any multi-key index conservatively blocks index order pushdown. Where it does
and per-path metadata is usable, a multi-key **equality prefix** may feed a
streaming `GroupAggregate` if every grouped/ordered path is scalar. A
multi-key group/order path remains unsafe and is rejected. All normal
equality-prefix and reduced-correlated soundness rules must also hold.

### Index-Only Scans Are Also Per Path

A composite index may be `isMultiKey: true` and still support an index-only
count/group (or covered find when that feature is enabled) when:

- per-path metadata is available and honored;
- every referenced filter, group, accumulator, and projection path is proven
  scalar; and
- no other fidelity blocker such as truncation requires a heap recheck.

Referencing a path listed in `multiKeyPaths`, or lacking usable per-path
metadata, blocks the index-only scan. An unreferenced multi-key path elsewhere
in the same index does **not** block it.

---

## numDuplicates and Dedup Overhead

`numDuplicates` in the explain output shows how many duplicate TIDs were encountered. High values indicate:
- Arrays in the indexed field are large
- Significant dedup work happening during scan
- More index entries scanned than unique documents

**Heuristic**: If `numDuplicates / totalKeysExamined > 0.5`, the multi-key overhead is substantial.

---

## Correlated Terms: Determine the Physical Layout First

A compound index over paths under the same array can have two different
physical layouts. The distinction is fundamental:

### Full-correlated (legacy cross-product) terms

The index stores cross-element combinations. For a document such as
`arr: [{a:1,b:9}, {a:5,b:2}]`, it can contain the composite pair `(a=1,b=2)`.
Therefore separate dotted predicates such as
`find({"arr.a":1, "arr.b":2})` may legitimately use both exact bounds and still
preserve the MongoDB API's cross-element matching semantics.

### Reduced-correlated terms

The index stores one composite term per array element, not the cross-product.
In extended explain, `hasCorrelatedTerms: true` identifies this reduced layout.
For separate dotted predicates, combining bounds from different elements would
miss valid documents, so unsafe secondary bounds must be pruned and rechecked.

The layout is fixed at **index build time**. If `hasCorrelatedTerms` is
absent, do not assume the index is full-correlated; check index metadata or
build history.

### Safe reduced-correlated `$elemMatch` bounds are query-time gated

A single `$elemMatch` owner requires its conditions to match one array element,
so its same-scope indexed fields are safe to co-bound on a reduced-correlated
index. The planner retains those fields only when:

- the service supports reduced-correlated bounds planning;
- the index has planner-readable per-path metadata; and
- the paths are in one supported `$elemMatch` scope.

If that support or the metadata is unavailable, execution falls back to
trim-to-leading behavior and secondary fields widen to `(MinKey, MaxKey)` with
a runtime recheck. This support is not visible in the explain output; **do not
assume it is on**.

With multiple `$elemMatch` clauses on one correlated prefix, only one owner
group can drive a single scan. When reduced-correlated bounds planning applies, the
selected owner's safe same-scope fields may co-bound; competing owners are rechecked.
This is **not** the same as “only one path can bound.”

### What to inspect

- `hasCorrelatedTerms: true` → reduced-correlated physical terms.
- `multiKeyPaths` present → planner-readable per-path metadata is available.
- Exact secondary bounds plus `rctBoundsPlanApplied` in raw plans → the
  reduced-correlated planning pass certified those bounds.
- Use explain/index metadata or build history to identify an existing
  index's layout and metadata.

### Index field ordering

Apply ESR after selecting the query shape that can actually drive the scan:

1. Most selective equality path from the driving `$elemMatch` owner first.
2. Remaining same-owner and non-correlated equality paths.
3. Sort/group-key paths.
4. Range paths.

For a legacy full-correlated index, separate same-array equalities can also
co-bound, but account for its larger cross-product storage/write cost.

---

## Diagnosing a Multi-Key Sort/Group Block

**Symptom**: `isMultiKey: true`, an explicit `SORT`/`HashAggregate` appears,
or a covered operation falls back to a heap `Index Scan`.

Check in this order:

1. Is `multiKeyPaths` present? If not, the index lacks usable per-path metadata
   and the planner must use conservative index-wide behavior.
2. Is the actual sort/group/target path listed in `multiKeyPaths`?
3. Is the filter on a multi-key sort path narrowing, or is it a harmless
   full-domain bound such as `$exists: true`?
4. Do ordinary ESR prefix/direction rules and any reduced-correlated
   soundness gates also pass?

---

## Per-Path Metadata and Legacy Fallback

Per-path tracking is supported; the old “entire index is multi-key, so scalar
trailing paths cannot push” limitation is stale.

However, the metadata is attached to the **index**. An index built without
planner-readable tracking remains conservative. Extended explain distinguishes the cases:

- `isMultiKey: true` plus `multiKeyPaths: ...` → use the listed paths.
- `isMultiKey: true` with no `multiKeyPaths` → path state is unknown; expect
  index-wide fallback until the index is rebuilt.

---

## Recommendations for Multi-Key Problems

1. **Verify metadata first**: Don't add a separate scalar index merely because `isMultiKey: true`; the existing index may already support the operation per path.
2. **Rebuild/reindex legacy metadata when justified**: If the index lacks `multiKeyPaths`, rebuild it to add current metadata before concluding the query needs a new index.
3. **If arrays are not needed**: Remove the array from the data model if it is always a single value.
4. **Keep arrays small**: Large arrays increase duplicate index entries and dedup overhead even when pushdown is legal.
5. **Use `$unwind` deliberately**: It can make grouping semantics explicit but adds row expansion and should be validated with an explain.
