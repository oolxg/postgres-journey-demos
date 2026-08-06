# 02. SELECT by id — plain Index Scan

Thesis: §4.3.3
Prerequisite: [00_setup.md](00_setup.md).

`id` is the pkey, unique. A point lookup hits exactly one TID — no bitmap reorder needed. Planner picks **Index Scan**: descend the btree, fetch the one matching heap tuple, return.

## Point lookup

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, COSTS OFF)
SELECT name, zipcode FROM person WHERE id = 5000;
```

```
 Index Scan using person_pkey on public.person (actual time=0.052..0.058 rows=1.00 loops=1)
   Output: name, zipcode
   Index Cond: (person.id = 5000)
   Index Searches: 1
   Buffers: shared hit=6
 Planning:
   Buffers: shared hit=102 dirtied=1
 Planning Time: 0.143 ms
 Execution Time: 0.083 ms
```

`hit=6` is higher than the apparent 3 unique pages (root + leaf + heap): EXPLAIN BUFFERS counts every `ReadBuffer` call, including re-pins across the `index_beginscan` / `index_getnext_tid` / `index_fetch_heap` boundary.

## Range scan

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, COSTS OFF)
SELECT name FROM person WHERE id BETWEEN 5000 AND 5010;
```

```
 Index Scan using person_pkey on public.person (actual time=0.018..0.024 rows=11.00 loops=1)
   Index Cond: ((person.id >= 5000) AND (person.id <= 5010))
   Index Searches: 1
   Buffers: shared hit=3
 Execution Time: 0.043 ms
```

11 sequential rows. Because `id` was inserted monotonically, all 11 live in the same heap block — heap buffers = 1 (and total = root + leaf + heap = 3).

## Source path

- Planner: `src/backend/optimizer/path/indxpath.c` builds an IndexPath, planner picks IndexScan when the predicate is highly selective (estimated 1 row)
- Executor: `src/backend/executor/nodeIndexscan.c:ExecIndexScan`
  → `src/backend/access/index/indexam.c:index_getnext_tid` (calls `amgettuple` → `btgettuple`)
  → `src/backend/access/index/indexam.c:index_fetch_heap`
  → `src/backend/access/heap/heapam_handler.c:heapam_index_fetch_tuple`
  → `src/backend/access/heap/heapam.c:heap_hot_search_buffer` (per-TID heap visit, walks the HOT chain)

---

← Previous: [01_select_zipcode.md](01_select_zipcode.md) | Next: [03_seq_scan_baseline.md](03_seq_scan_baseline.md) →
