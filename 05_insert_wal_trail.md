# 05. INSERT WAL trail — four records, three of them mostly page image

Thesis: §4.4.1
Prerequisite: [00_setup.md](00_setup.md) (74 heap pages, 10000 rows).

A single-row INSERT emits exactly four WAL records: the heap tuple, one index entry per index, and the commit. The interesting part is their size — 14 258 bytes for a 50-byte row — and where those bytes go.

## Setting up a clean window

Two LSN readings bracket a *span of time*, not a statement, so anything another backend logs in that span also lands in the window. Two record types show up this way and are easy to mistake for INSERT's own work:

- `XLOG / FPI_FOR_HINT` — emitted when a **read** is the first to touch a page after a checkpoint and sets `HEAP_XMIN_COMMITTED` hint bits (`bufmgr.c:MarkBufferDirtyHint`).
- `Heap2 / PRUNE_ON_ACCESS` — the opportunistic prune a **scan** performs in passing (`pruneheap.c`, reached only from `heap_page_prune_opt`). No INSERT path calls it.

Reading the table once before the checkpoint sets every hint bit in advance, which keeps both out of the window:

```sql
SELECT count(*) FROM person;   -- set all hint bits now
CHECKPOINT;                    -- reset the FPI bookkeeping
```

## Capture

```sql
-- pg_current_wal_lsn() returns the current WAL write position (a byte offset into
-- the WAL stream). \gset is a psql-only directive: capture the single result
-- into a client-side variable.
SELECT pg_current_wal_lsn() AS lsn_before \gset

INSERT INTO person VALUES (10001, 'NewPerson', '10042', 30);

SELECT pg_current_wal_lsn() AS lsn_after \gset

SELECT resource_manager, record_type, record_length
FROM pg_get_wal_records_info(:'lsn_before', :'lsn_after')
ORDER BY start_lsn;
```

```
 resource_manager | record_type | record_length
------------------+-------------+---------------
 Heap             | INSERT      |          4458
 Btree            | INSERT_LEAF |          2473
 Btree            | INSERT_LEAF |          7293
 Transaction      | COMMIT      |            34
```

Four records, 14 258 bytes total. Two `INSERT_LEAF` records confirm that each index gets its own entry on every INSERT, regardless of HOT eligibility — Demo 07 is the contrast case.

## Where the bytes are

`pg_get_wal_block_info` splits each record into its page image and everything else:

```sql
SELECT relfilenode, relblocknumber AS blk, record_type,
       record_length AS record, block_fpi_length AS image,
       record_length - block_fpi_length AS rest, block_fpi_info
FROM pg_get_wal_block_info(:'lsn_before', :'lsn_after')
ORDER BY start_lsn, block_id;
```

```
 relfilenode | blk | record_type | record | image | rest |  block_fpi_info
-------------+-----+-------------+--------+-------+------+------------------
       25628 |  73 | INSERT      |   4458 |  4404 |   54 | {HAS_HOLE,APPLY}
       25637 |  29 | INSERT_LEAF |   2473 |  2420 |   53 | {HAS_HOLE,APPLY}
       25639 |   9 | INSERT_LEAF |   7293 |  7240 |   53 | {HAS_HOLE,APPLY}
```

(relfilenode values differ per run; 25628 = `person`, 25637 = `person_pkey`, 25639 = `person_zipcode_idx`.)

53–54 bytes per record is the actual change description. Everything else is a full-page image, written because this is the first time each page is modified in this checkpoint cycle.

## Why the three images differ in size

`HAS_HOLE` is the clue. A page keeps its free space in one contiguous run between the line pointers and the tuple data, and the image omits that run. So an image costs 8192 minus the free space on the page. The page header states both boundaries, so the arithmetic is checkable:

```sql
SELECT 'person/73' AS page, lower, upper, (upper-lower) AS hole,
       8192-(upper-lower) AS predicted_image
FROM page_header(get_raw_page('person', 73))
UNION ALL SELECT 'pkey/29', lower, upper, (upper-lower), 8192-(upper-lower)
  FROM page_header(get_raw_page('person_pkey', 29))
UNION ALL SELECT 'zipcode/9', lower, upper, (upper-lower), 8192-(upper-lower)
  FROM page_header(get_raw_page('person_zipcode_idx', 9));
```

```
   page    | lower | upper | hole | predicted_image
-----------+-------+-------+------+-----------------
 person/73 |   316 |  4104 | 3788 |            4404
 pkey/29   |   500 |  6272 | 5772 |            2420
 zipcode/9 |   160 |  1112 |  952 |            7240
```

`predicted_image` matches `block_fpi_length` in all three rows. The zipcode leaf is the expensive one (7240 B) because deduplication packs it full, leaving only 952 bytes of slack. The pkey leaf, half empty after a split, costs 2420.

## The same statement again, same checkpoint cycle

Note: the *first* repeat often catches a burst of `FPI_FOR_HINT` and `PRUNE_ON_ACCESS`
records from autovacuum touching the system catalogs for the first time after the
checkpoint. They are not ours. Running the insert once more gives a clean window,
and the numbers below are stable from the third insert onward.

```sql
INSERT INTO person VALUES (10002, 'NewPerson', '10042', 30);   -- absorbs the catalog noise

SELECT pg_current_wal_lsn() AS l3 \gset
INSERT INTO person VALUES (10003, 'NewPerson', '10042', 30);
SELECT pg_current_wal_lsn() AS l4 \gset

SELECT resource_manager, record_type, record_length
FROM pg_get_wal_records_info(:'l3', :'l4') ORDER BY start_lsn;
```

```
 resource_manager | record_type | record_length
------------------+-------------+---------------
 Heap             | INSERT      |            81
 Btree            | INSERT_LEAF |            64
 Btree            | INSERT_LEAF |            64
 Transaction      | COMMIT      |            34
```

243 bytes against 14 258 — **58× less** for identical work. The images are paid once per page per checkpoint cycle, not once per row.

This is why record **counts** in this repository are stable between runs while record **sizes** are not: the sizes depend on where in the checkpoint cycle the capture falls.

## Why an image at all

A crash during a page write can leave the page half old and half new, with a checksum matching neither. Recovery cannot patch such a page. An image does not patch — it overwrites the page with a copy known to be whole (`BLK_RESTORED`), after which the small delta records apply cleanly. Recovery skips records the page has already absorbed by comparing the page's stored LSN (`xlogutils.c:420`, `BLK_DONE`).

## Heap state after

```sql
SELECT pg_relation_size('person') / 8192 - 1 AS last_block;
--  last_block
-- ------------
--          73

SELECT lp, lp_off, lp_flags, lp_len, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('person', 73))
ORDER BY lp DESC LIMIT 3;
```

```
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_ctid
----+--------+----------+--------+--------+--------+---------
 73 |   4104 |        1 |     50 |    772 |      0 | (73,73)
 72 |   4160 |        1 |     54 |    769 |      0 | (73,72)
 71 |   4216 |        1 |     52 |    769 |      0 | (73,71)
```

The new tuple landed on page 73, item 73. `t_ctid = (73,73)` — self pointer. Page count unchanged (74) — there was room on page 73. Note `lp_off = 4104` for the newest tuple: that is the `pd_upper` used in the image arithmetic above.

Both indexes had room without growing — `person_pkey` still 30 pages, `person_zipcode_idx` still 11.

## Source path

- `src/backend/executor/nodeModifyTable.c:ExecInsert` → `heapam.c:heap_insert` → `XLogInsert` with `XLOG_HEAP_INSERT`
- Index entries: `execIndexing.c:ExecInsertIndexTuples` → `nbtree.c:btinsert` → `nbtinsert.c:_bt_insertonpg` (`XLOG_BTREE_INSERT_LEAF`)
- Full-page image decision and the hole: `src/backend/access/transam/xloginsert.c:XLogRecordAssemble`, flags in `src/include/access/xlogrecord.h` (`BKPIMAGE_HAS_HOLE`)
- Commit flush: `src/backend/access/transam/xact.c:1502` (`XLogFlush`)
- Replay: `src/backend/access/transam/xlogutils.c:420` (`BLK_DONE` vs `BLK_NEEDS_REDO`)

---

← Previous: [04b_cold_cache.md](04b_cold_cache.md) | Next: [06_page_extension.md](06_page_extension.md) →
