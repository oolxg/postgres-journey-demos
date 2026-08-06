# 14. Index-Only Scan via visibility map — `Heap Fetches: 0`

Thesis: §4.3.6
Prerequisite: [00_setup.md](00_setup.md), then run Demo 13 (or any large workload) and `VACUUM person;` so the VM marks every page all-visible.

When VM bit `all_visible` is set for a heap page, an index scan can answer queries from the index alone, skipping heap fetches. EXPLAIN's `Heap Fetches:` counter increments only when the executor falls back to a heap visit (VM bit not set). After VACUUM marks every page all-visible, a covered query (output column lives in the index) returns 0 heap fetches.

## Setup: refresh VM with VACUUM

```sql
VACUUM person;

-- pg_visibility_map: one row per heap page, (all_visible, all_frozen) bits.
SELECT all_visible, count(*) AS pages
FROM pg_visibility_map('person')
GROUP BY all_visible
ORDER BY all_visible;
```

```
 all_visible | pages
-------------+-------
 t           |  1863      <-- ALL pages all-visible (after Demo 13's 200k inserts)
```

All 1863 heap pages are all-visible.

## Covered query — Heap Fetches: 0

```sql
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF)
SELECT zipcode FROM person WHERE zipcode = '10063';
```

```
 Index Only Scan using person_zipcode_idx on person
   Index Cond: (zipcode = '10063'::text)
   Heap Fetches: 0                          <-- zero heap fetches
   Buffers: shared hit=3
 Execution Time: 0.249 ms
```

The output column `zipcode` is in `person_zipcode_idx` → executor returns it from the index entry without consulting the heap. All 3 buffer hits are the index root, one leaf, and the VM page (the metapage is served from the planner’s `rd_amcache` copy and never re-pinned).

## Compare with uncovered column

```sql
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF)
SELECT name FROM person WHERE zipcode = '10063';
```

```
 Bitmap Heap Scan on person
   Heap Blocks: exact=74
   Buffers: shared hit=76                   <-- 74 heap pages + 2 index pages
   ->  Bitmap Index Scan on person_zipcode_idx
 Execution Time: 0.272 ms
```

Same predicate, same number of returned rows (100). The covered-column query touched **3 buffers**; the uncovered-column query touched **76**. On warm cache the gap is small (0.249 ms vs 0.272 ms — both queries' pages already in the pool). On cold cache the difference balloons: 76 disk reads cost ~1.5 ms additional execution time. The bigger the relation and the colder the cache, the more decisive the IOS win.

## How it works

During the index scan, for each TID found in the leaf, the executor checks the visibility map for the TID's heap page (via `VM_ALL_VISIBLE` test). If set, the heap fetch is skipped and the index tuple's data is returned directly. If unset, the executor falls back to a heap fetch to verify visibility — which is why VM staleness is safe (just slightly more I/O), not a correctness bug. EXPLAIN's `Heap Fetches:` counter increments only on the fall-back branch; after VACUUM marked all pages all-visible, every TID hit the fast path and `Heap Fetches: 0`.

Why VM check is **not** a heap fetch: VM is a separate fork (`relforknumber=2`), a tiny side-file — one VM page covers ~32k heap pages. `Heap Fetches:` counts main-fork reads only (`relforknumber=0`).

## Two VM bits, three valid states

```sql
-- pg_visibility shows VM and on-page bits side by side for a specific page.
SELECT * FROM pg_visibility('person', 73);
```

```
 all_visible | all_frozen | pd_all_visible
-------------+------------+----------------
 t           | f          | t
```

Three columns:
- `all_visible` — VM bit
- `all_frozen` — VM bit; implies `all_visible` (frozen ⇒ visible). So three valid states: `(f,f)`, `(t,f)`, `(t,t)`; never `(f,t)`.
- `pd_all_visible` — duplicate on-page flag, kept in sync with the VM bit. A mismatch indicates corruption.

Any DML on the page clears both VM bits. VACUUM sets them lazily (and freezes when xmin is old enough). See thesis §4.5.3 for the bit lifecycle.

## Source path

- `src/backend/executor/nodeIndexonlyscan.c:IndexOnlyNext`
  - `VM_ALL_VISIBLE` macro (`src/include/access/visibilitymap.h`) — pin VM page, read 2 bits, return
  - On YES: return data from the index tuple; do not call `index_fetch_heap`
  - On NO: fall back to `src/backend/access/index/indexam.c:index_fetch_heap` and increment `Heap Fetches`
- VM mechanics: `src/backend/access/heap/visibilitymap.c`
  - `visibilitymap_set` — sets bits during VACUUM (`lazy_scan_heap`)
  - `visibilitymap_clear` — clears bits inside `heap_insert` / `heap_update` / `heap_delete` (every DML)

---

← Previous: [13_page_split.md](13_page_split.md) | Back to [README](README.md)
