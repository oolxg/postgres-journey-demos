# 04b. Server restart — cold cache

Thesis: §4.2.5
Prerequisite: [00_setup.md](00_setup.md). Run after Demo 04 (so the pool is warm with `person` pages) for the clearest contrast. Needs shell access to `pg_ctl` to restart the server between phases.

PG keeps a shared buffer pool in memory. Restarting the server flushes this pool. The on-disk files (heap, indexes, WAL, catalogs) survive — but on the next query, every page must be re-read from disk.

## Buffer pool before restart

```sql
-- Per-relation buffer count for our three relations.
SELECT c.relname, count(*) AS buffers
FROM pg_buffercache b
JOIN pg_class c ON c.relfilenode = b.relfilenode
WHERE c.relname IN ('person','person_pkey','person_zipcode_idx')
GROUP BY c.relname
ORDER BY c.relname;
```

```
        name        | buffers
--------------------+---------
 person             |      78    -- all 74 heap pages + a few prior versions
 person_pkey        |      30    -- entire index
 person_zipcode_idx |      11    -- entire index
```

## Restart the server

```bash
./inst/bin/pg_ctl -D ./data restart -m fast
```

## Buffer pool immediately after restart (before any query)

Re-run the same buffer-pool query in a fresh session:

```
 name | buffers
------+---------
(0 rows)
```

Zero buffers for `person` and its indexes. The pool was wiped.

## First query after restart — cold reads

```sql
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF)
SELECT name FROM person WHERE zipcode = '10063';
```

```
 Bitmap Heap Scan on person (actual time=0.511..1.637 rows=100.00 loops=1)
   Heap Blocks: exact=74
   Buffers: shared read=76 dirtied=1            <-- "read", not "hit"
   ->  Bitmap Index Scan on person_zipcode_idx
         Buffers: shared read=2                  <-- "read"
 Planning:
   Buffers: shared hit=82 read=12
 Planning Time: 1.903 ms
 Execution Time: 1.978 ms
```

Note `shared read=76` and `read=2` — the executor pulled 78 pages from disk. Planning hit 82 pages from cache (catalog pages partially survive as they were loaded by the connection's startup) plus 12 read from disk.

Buffer pool after the cold query (extended to show block numbers):

```sql
SELECT c.relname,
       count(*) AS buffers,
       string_agg(b.relblocknumber::text, ',' ORDER BY b.relblocknumber) AS blocks
FROM pg_buffercache b
JOIN pg_class c ON c.relfilenode = b.relfilenode
WHERE c.relname IN ('person','person_pkey','person_zipcode_idx')
GROUP BY c.relname
ORDER BY c.relname;
```

```
        name        | buffers | blocks
--------------------+---------+--------
 person             |      74 | 0..73         -- all heap pages now loaded
 person_pkey        |       1 | 0             -- only metapage; query used the other index
 person_zipcode_idx |       3 | 0,3,10        -- metapage + root (3) + one leaf (10)
```

`person_pkey` got 1 buffer — its metapage. The query used `person_zipcode_idx`, but the planner still consulted `pg_index` for `person_pkey` to compute alternate paths; that opens the relation, which loads its metapage into the pool.

`person_zipcode_idx` got 3 buffers — metapage (0), root (3), and leaf for `'10063'` (10). Btree's descent path: read meta → follow `fastroot` to root page 3 → binary search → fetch leaf 10.

## Same query again — now warm

```sql
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF)
SELECT name FROM person WHERE zipcode = '10063';
```

```
 Bitmap Heap Scan on person (actual time=0.030..0.112 rows=100.00 loops=1)
   Buffers: shared hit=76                        <-- "hit", no disk
   ->  Bitmap Index Scan on person_zipcode_idx
         Buffers: shared hit=2
 Planning Time: 0.070 ms                         <-- 1.903 ms -> 0.070 ms
 Execution Time: 0.290 ms                        <-- 1.978 ms -> 0.290 ms
```

All `shared hit`, no `read`. Cold-to-warm speedup: **execution 7x faster, planning 27x faster** for this query. The cold cost is dominated by 78 disk reads at ~20 µs each, totalling ~1.5 ms — roughly the gap.

This is the main reason production deployments often use `pg_prewarm` after a restart, or run a known-large query once to populate the pool, before letting real traffic in.

## Source path

- Shared buffer pool initialisation: `src/backend/storage/buffer/buf_init.c:InitBufferPool`
- Cluster startup loads only system catalogs into the pool; user-relation pages are loaded lazily on first access
- `pg_prewarm` contrib (`contrib/pg_prewarm/`) provides `pg_prewarm(regclass, mode, fork, first, last)` to populate the pool deliberately
- `BufferAccessStrategy` (`src/backend/storage/buffer/freelist.c`) is bypassed for normal heap reads — they hit the default strategy and bump usagecount to 1, then climb on repeats (see Demo 04)

---

← Previous: [04_buffer_pool_walkthrough.md](04_buffer_pool_walkthrough.md) | Next: [05_insert_wal_trail.md](05_insert_wal_trail.md) →
