# 04. Buffer pool contents via pg_buffercache

Thesis: §4.2.4
Prerequisite: [00_setup.md](00_setup.md).

`EXPLAIN BUFFERS` counts I/O against the executor; it does not show *what is resident* in the shared pool right now. `pg_buffercache` exposes the pool slot by slot. Each row is one shared-buffer slot with these fields of interest:

- `bufferid` — slot index in `BufferDescriptors[]` (0..NBuffers-1, default 16384). Just an address inside the pool; shifts between runs.
- `relfilenode`, `relforknumber`, `relblocknumber` — which page lives in this slot (NULL for empty slots). `relfilenode` joins `pg_class.relfilenode`; `relforknumber` selects the fork (`0` = main, `1` = FSM, `2` = visibility map); `blk=0` of an index file is the metapage.
- `isdirty` — `t` if the page was modified in memory and not yet flushed.
- `usagecount` — clock-sweep counter, 0..5, bumped on each pin, capped at `BM_MAX_USAGE_COUNT = 5`. Eviction prefers `usagecount = 0` slots.
- `pinning_backends` — number of backends currently holding the page pinned. Usually `0` between queries.

## Cold inspection — one Bitmap Heap Scan after `evict_all`

```sql
-- pg_buffercache_evict_all is PG 18+. It walks all slots and evicts every page
-- (flushing dirty ones to disk first), giving us a clean baseline.
SELECT * FROM pg_buffercache_evict_all();

SELECT name FROM person WHERE zipcode = '10063';

-- Join pg_buffercache against pg_class to filter by relation name. The
-- alternative — WHERE on a function-scan column of pg_buffercache_pages()
-- directly — breaks on this build's instrumented planner ("bogus varno").
SELECT b.bufferid,
       b.relfilenode,
       b.relforknumber AS fork,
       b.relblocknumber AS blk,
       b.isdirty AS dirty,
       b.usagecount AS uc,
       b.pinning_backends AS pin
FROM pg_buffercache b
JOIN pg_class c ON c.relfilenode = b.relfilenode
WHERE c.relname IN ('person','person_pkey','person_zipcode_idx')
ORDER BY c.relname, b.relforknumber, b.relblocknumber;
```

```
 bufferid | relfilenode | fork | blk | dirty | uc | pin
----------+-------------+------+-----+-------+----+-----
     8223 |       16821 |    0 |   0 |   f   |  1 |   0    -- pkey metapage
     8224 |       16823 |    0 |   0 |   f   |  1 |   0    -- zipcode_idx metapage
     8246 |       16823 |    0 |   3 |   f   |  1 |   0    -- zipcode_idx leaf
     8247 |       16823 |    0 |  10 |   f   |  1 |   0    -- zipcode_idx leaf
     8248 |       16812 |    0 |   0 |   f   |  1 |   0    -- person heap blk 0
     8249 |       16812 |    0 |   1 |   f   |  1 |   0
     8250 |       16812 |    0 |   2 |   f   |  1 |   0
       …  (74 person heap rows total, blk 0..73, all uc=1, dirty=f, pin=0)
     8320 |       16812 |    0 |  72 |   f   |  1 |   0
     8321 |       16812 |    0 |  73 |   f   |  1 |   0
(78 rows)
```

How to read this output, row by row:

- The two metapage rows (`16821 blk=0`, `16823 blk=0`) are the per-index headers — `_bt_getroot` reads them once to find the root.
- The `16823 blk=3, blk=10` rows are the root (`blk=3`) and one leaf (`blk=10`) of `person_zipcode_idx`. The Bitmap Index Scan descended metapage → root (`blk=3`) → leaf (`blk=10`), where the posting tuples for `'10063'` live.
- The 74 consecutive `16812 blk=0..73` rows are the entire heap. `zipcode='10063'` matches ~100 rows scattered (≈1.4 hits per page), so Bitmap Heap Scan ends up touching every page in block order. Bufferids 8248..8321 are consecutive because `evict_all` left those slots free and bufmgr handed them out in order.
- `isdirty=f` for the whole 78 rows: read-only query, hint bits already on disk from populate-time ANALYZE (`evict_all` flushed those pages on the way out).
- `usagecount=1` everywhere: each page pinned exactly once during this single scan.
- `pinning_backends=0` everywhere: no concurrent backend holds these pages, and our own session has finished pinning by the time `pg_buffercache` runs.

The same 78 pages show up in `EXPLAIN BUFFERS` for this query as `shared hit=76` (Bitmap Heap Scan, re-pin double-counting on the heap side) plus `shared hit=2` (Bitmap Index Scan). EXPLAIN gives executor counters; `pg_buffercache` shows the exact pages those counters represent.

## Warm inspection — repeat five more times

```sql
SELECT name FROM person WHERE zipcode = '10063';
SELECT name FROM person WHERE zipcode = '10063';
SELECT name FROM person WHERE zipcode = '10063';
SELECT name FROM person WHERE zipcode = '10063';
SELECT name FROM person WHERE zipcode = '10063';

SELECT c.relname,
       b.relblocknumber AS blk,
       b.isdirty AS dirty,
       b.usagecount AS uc
FROM pg_buffercache b
JOIN pg_class c ON c.relfilenode = b.relfilenode
WHERE c.relname IN ('person','person_pkey','person_zipcode_idx')
ORDER BY c.relname, b.relblocknumber;
```

```
 relname            | blk | dirty | uc
--------------------+-----+-------+----
 person             | 0   |   f   |  5
 person             | 1   |   f   |  5
 ...                                (74 person heap rows, all uc=5)
 person             | 73  |   f   |  5
 person_pkey        | 0   |   f   |  1    <-- metapage stays at uc=1
 person_zipcode_idx | 0   |   f   |  1    <-- metapage stays at uc=1
 person_zipcode_idx | 3   |   f   |  5    <-- leaf climbed to uc=5
 person_zipcode_idx | 10  |   f   |  5    <-- leaf climbed to uc=5
```

The clock-sweep counter saturates at 5 for buffers touched on every scan. Metapages stay at 1 because `_bt_getroot` (`nbtree/nbtpage.c:344`) caches `BTMetaPageData` in `rel->rd_amcache` after the first descent — subsequent scans skip the metapage entirely and jump straight to the cached `fastroot`, so the metapage buffer is never re-pinned. The leaves we use on every scan climb to the maximum.

## Aggregate views

```sql
SELECT * FROM pg_buffercache_summary();
SELECT * FROM pg_buffercache_usage_counts() ORDER BY usage_count;
```

```
 buffers_used | buffers_unused | buffers_dirty | buffers_pinned | usagecount_avg
--------------+----------------+---------------+----------------+----------------
          190 |          16194 |             5 |              0 |            3.5

 usage_count | buffers | dirty | pinned
-------------+---------+-------+--------
           0 |   16182 |     0 |      0
           1 |      42 |     0 |      0
           2 |      29 |     5 |      0
           3 |      17 |     0 |      0
           4 |       2 |     0 |      0
           5 |     112 |     2 |      0
```

190 of 16384 buffers used (default `shared_buffers` = 128 MB / 8 KB). 5 dirty pages — catalog hint-bits set during connection startup. `pinning_backends=0` for the whole pool because no query is active at the moment of inspection.

## Source path

- View / SRF: `contrib/pg_buffercache/pg_buffercache_pages.c` walks `BufferDescriptors[]` and returns one row per slot
- Slot state: `BufferDesc.state` (`src/include/storage/buf_internals.h`) packs `usagecount`, pin count, and flag bits (`BM_DIRTY`, `BM_VALID`, …)
- usagecount bump: `src/backend/storage/buffer/bufmgr.c:PinBuffer` (default-strategy path, the `buf_state += BUF_USAGECOUNT_ONE` line, capped at `BM_MAX_USAGE_COUNT = 5`). `IncrBufferRefCount` and `PinBuffer_Locked` deliberately do not touch usagecount. `BufferAccessStrategy` rings (seq scans, VACUUM) lift it from 0 to 1 only — ring-buffered pages never climb to 5

---

← Previous: [03_seq_scan_baseline.md](03_seq_scan_baseline.md) | Next: [04b_cold_cache.md](04b_cold_cache.md) →
