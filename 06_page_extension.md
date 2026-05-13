# 06. Page extension — heap file grows when no free space remains

Thesis: §4.3.2
Prerequisite: [00_setup.md](00_setup.md) (74 heap pages, 10000 rows).

Heap is a file on disk split into 8 KB pages. When INSERT needs free space, `heap_insert` consults the FSM (Free Space Map): "do you know any page with at least N free bytes?" If the FSM returns `InvalidBlockNumber`, control falls into `RelationAddBlocks`, which calls `ExtendBufferedRelBy` → `smgrzeroextend` to grow the file. Each newly initialised page emits one `Heap / INSERT+INIT` WAL record (`XLOG_HEAP_INSERT` with the `XLOG_HEAP_INIT_PAGE` flag).

## Bulk insert that overflows

```sql
-- Heap pages before. pg_relation_size returns bytes; divide by 8192 (page size).
SELECT pg_relation_size('person') / 8192 AS heap_pages_before;
--  heap_pages_before
-- -------------------
--                 74

-- Capture WAL LSN before the bulk insert so we can read every record it emits.
SELECT pg_current_wal_lsn() AS lsn_before \gset

-- 1000 rows. Last existing page (73) absorbs the first ~50; after it fills,
-- FSM exhausts and heap_insert calls into the extension path for each new page.
INSERT INTO person (id, name, zipcode, age)
SELECT g, 'Filler_' || g, (10000 + g % 100)::text, (20 + g % 60)::smallint
FROM generate_series(11001, 12000) g;

SELECT pg_current_wal_lsn() AS lsn_after \gset

SELECT pg_relation_size('person') / 8192 AS heap_pages_after;
--  heap_pages_after
-- ------------------
--                81

SELECT resource_manager, record_type, count(*) AS cnt
FROM pg_get_wal_records_info(:'lsn_before', :'lsn_after')
GROUP BY 1, 2
ORDER BY 1, 2;
```

```
 resource_manager |  record_type  | cnt
------------------+---------------+------
 Btree            | DEDUP         |   28
 Btree            | INSERT_LEAF   | 1997
 Btree            | INSERT_UPPER  |    3
 Btree            | SPLIT_L       |    1
 Btree            | SPLIT_R       |    2
 Heap             | INSERT        |  993
 Heap             | INSERT+INIT   |    7      <- one record per newly extended page
 Standby          | RUNNING_XACTS |    1
 Transaction      | COMMIT        |    1
```

Heap grew by 7 pages (74 → 81); the WAL trail shows 7 `Heap / INSERT+INIT` records, one per page extension. `INSERT+INIT` is `XLOG_HEAP_INSERT` with the `XLOG_HEAP_INIT_PAGE` flag set: `PageInit` zeroes the new page header before the first tuple goes in, both written in a single WAL record. Plain `INSERT` records (993 of them) cover rows that fit pages already in the file.

The btree side reacted too: 1997 `INSERT_LEAF` (≈ 2 indexes × 1000 rows minus dedup absorption), 28 `DEDUP` passes on `person_zipcode_idx`, three leaf splits (`SPLIT_L`/`SPLIT_R`) and three resulting `INSERT_UPPER` records propagating new separators upward. Heap extension and index growth are independent. Each AM extends its own files through the buffer manager's `ExtendBufferedRel(By)` family (heap calls `ExtendBufferedRelBy` directly from `RelationAddBlocks`; btree calls the singular `ExtendBufferedRel` wrapper from `_bt_allocbuf`, which forwards to `ExtendBufferedRelBy` with `extend_by=1`).

## Buffer pool around the extension boundary

```sql
SELECT b.relblocknumber AS blk,
       b.isdirty AS dirty,
       b.usagecount AS uc
FROM pg_buffercache b
JOIN pg_class c ON c.relfilenode = b.relfilenode
WHERE c.relname = 'person'
  AND b.relforknumber = 0
  AND b.relblocknumber BETWEEN 70 AND 80
ORDER BY b.relblocknumber;
```

```
 blk | dirty | uc
-----+-------+----
  70 |   f   |  5     <- pre-existing, full, not touched in this run
  71 |   f   |  5
  72 |   f   |  5
  73 |   t   |  5     <- last existing page, took the first inserts
  74 |   t   |  5     <- new page #1 (XLOG_HEAP_INSERT+INIT emitted here)
  75 |   t   |  5
  76 |   t   |  5
  77 |   t   |  5
  78 |   t   |  5
  79 |   t   |  5
  80 |   t   |  5     <- new page #7
```

Pages 73–80 are dirty, every one at `usagecount = 5` (each took many inserts in quick succession; bufmgr capped the counter). The clean older pages (70–72) sit at `uc=5` from earlier SELECT scans and stay clean because no insert needed them.

## Source path on the extension branch

- `src/backend/executor/nodeModifyTable.c:ExecInsert` → `heapam.c:heap_insert` → `hio.c:RelationGetBufferForTuple`
- Loop over candidate blocks via `GetPageWithFreeSpace` (FSM lookup, `src/backend/storage/freespace/freespace.c`); if no block fits, fall through
- `hio.c:RelationAddBlocks` → `bufmgr.c:ExtendBufferedRelBy` (acquires the relation extension lock for non-temporary relations, calls `smgrzeroextend` to grow the relfilenode file by one or more zeroed pages)
- New buffer returned; `heap_insert` calls `PageInit`, places the tuple, writes the WAL record with `XLOG_HEAP_INIT_PAGE` flag, marks the buffer dirty
- After `RelationAddBlocks` returns, the new free space is recorded in the FSM via `RecordPageWithFreeSpace` (`freespace.c:194`) so the next insert finds it without another extension

---

← Previous: [05_insert_wal_trail.md](05_insert_wal_trail.md) | Next: [07_hot_update.md](07_hot_update.md) →
