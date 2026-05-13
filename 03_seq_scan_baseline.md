# 03. Seq Scan baseline — no index on `age`

Thesis: §4.2.3
Prerequisite: [00_setup.md](00_setup.md). Run after Demos 01 and 02 in the same session to see the cold→warm catcache transition.

`age` is not indexed. The planner has only one path: full **Seq Scan** over all 74 heap pages. This is the cost baseline against which the index-based plans in Demos 01 and 02 are measured.

## Plan and buffers

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, COSTS OFF)
SELECT count(*) FROM person WHERE age > 75;
```

```
 Aggregate (actual time=5.347..5.348 rows=1.00 loops=1)
   ->  Seq Scan on public.person (actual time=0.073..5.187 rows=664.00 loops=1)
         Filter: (person.age > 75)
         Rows Removed by Filter: 9336
         Buffers: shared hit=74
 Planning:
   Buffers: shared hit=12
 Planning Time: 0.415 ms
 Execution Time: 5.980 ms
```

- **Heap buffers** `hit=74` — every heap page visited (no index → no skip).
- **Planning buffers** `hit=12` — much smaller than the 96 hits Demo 01 paid because the catcache is now warm: `pg_class`, `pg_attribute`, etc. were loaded by the earlier queries. The remaining 12 reads are for catalogs specific to this query (smallint operator/proc, `age` column's `pg_statistic` entry).

664 rows match: `age = 20 + g % 60` gives ages in 20..79, so `age > 75` keeps ages 76, 77, 78, 79 — 4 buckets out of 60 ≈ 6.67% of 10000 ≈ 667.

## Source path

- Planner: no IndexScan path is even built (no index on `age`); SeqScan is the only candidate
- Executor: `src/backend/executor/nodeSeqscan.c:ExecSeqScan`
  → `src/backend/access/heap/heapam.c:heap_getnextslot`
  → `heapgettup_pagemode` walks the relation block by block, returning each visible tuple

---

← Previous: [02_select_id_point.md](02_select_id_point.md) | Next: [04_buffer_pool_walkthrough.md](04_buffer_pool_walkthrough.md) →
