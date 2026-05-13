# 05. INSERT WAL trail — six records per one-row INSERT

Thesis: §4.3.1
Prerequisite: [00_setup.md](00_setup.md) (74 heap pages, 10000 rows).

A single-row INSERT produces six WAL records. Captured by `pg_walinspect` between two LSNs.

## Capture

```sql
-- pg_current_wal_lsn() returns current WAL write position (a byte offset into
-- the WAL stream). \gset is a psql-only directive: capture the single result
-- into a client-side variable named lsn_before.
SELECT pg_current_wal_lsn() AS lsn_before \gset

INSERT INTO person VALUES (10001, 'NewPerson', '10042', 30);

SELECT pg_current_wal_lsn() AS lsn_after \gset

-- pg_get_wal_records_info reads every WAL record between two LSNs and returns
-- one row per record. :'lsn_before' is psql variable substitution (quoted).
-- Function lives in contrib/pg_walinspect.
SELECT start_lsn, resource_manager, record_type, record_length
FROM pg_get_wal_records_info(:'lsn_before', :'lsn_after')
ORDER BY start_lsn;
```

```
 start_lsn | resource_manager |   record_type   | record_length
-----------+------------------+-----------------+---------------
 0/1FB6B18 | XLOG             | FPI_FOR_HINT    |          8005
 0/1FB8A78 | Heap2            | PRUNE_ON_ACCESS |            52
 0/1FB8AB0 | Heap             | INSERT          |          4458
 0/1FB9C20 | Btree            | INSERT_LEAF     |          2473
 0/1FBA5E8 | Btree            | INSERT_LEAF     |          7293
 0/1FBC280 | Transaction      | COMMIT          |            34
```

One row, six WAL records:

1. **`XLOG / FPI_FOR_HINT`** (8005 bytes). When the INSERT opened heap page 73, visibility checks on the existing tuples set `HEAP_XMIN_COMMITTED` hint bits, dirtying the page. Hint-bit changes are normally not logged — they're a re-derivable optimisation — but with `data_checksums` (or `wal_log_hints`) enabled, a torn write of a partially-flushed page would leave a checksum mismatch on disk that recovery cannot fix without the original content. So the first time a page is dirtied in a checkpoint cycle for hint bits only, Postgres emits a stand-alone full page image. Source: `bufmgr.c:MarkBufferDirtyHint`.
2. **`Heap2 / PRUNE_ON_ACCESS`** (52 bytes). Opportunistic page prune triggered during the buffer pin — small dead-tuple cleanup riding along.
3. **`Heap / INSERT`** (4458 bytes). The actual heap tuple. Includes a full page image (post-prune state).
4. **`Btree / INSERT_LEAF`** (2473 bytes). Insert into `person_pkey`. Just the new index tuple — no FPI needed, the leaf page had already been imaged in this checkpoint cycle.
5. **`Btree / INSERT_LEAF`** (7293 bytes). Insert into `person_zipcode_idx`. Includes a full page image of the leaf — first time this leaf is written in this checkpoint cycle.
6. **`Transaction / COMMIT`** (34 bytes). Xact commit.

The two `INSERT_LEAF` records confirm: each index gets its own update on every INSERT, regardless of HOT.

## Heap state after

`pageinspect`'s `heap_page_items` decodes a single heap page, one row per line pointer. Last heap page = `pg_relation_size('person') / 8192 - 1`:

```sql
-- Derive last heap block: relation size in bytes / page size - 1 (0-indexed).
SELECT pg_relation_size('person') / 8192 - 1 AS last_block;
--  last_block
-- ------------
--          73

-- get_raw_page returns the raw 8 KB bytes; heap_page_items decodes it into
-- one row per line pointer (lp + tuple header fields). pageinspect contrib.
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

The new tuple landed on page 73, item 73. `t_xmin = 772` — the INSERT's xid (different from `xmin = 769` of populator rows). `t_ctid = (73,73)` — self pointer. Page count unchanged (74) — there was room on page 73.

Both indexes had room for the new entry without growing — `person_pkey` still 30 pages, `person_zipcode_idx` still 11 pages. The btree leaf for the new key (id=10001 → rightmost leaf; zipcode='10042' → some interior leaf) had spare space.

## Source path

- Heap insert: `src/backend/executor/nodeModifyTable.c:ExecInsert` → `src/backend/access/heap/heapam.c:heap_insert` → page pin, FSM lookup, tuple write, WAL emission, hint bits
- Index dispatch: `src/backend/executor/execIndexing.c:ExecInsertIndexTuples` walks `ResultRelInfo->ri_IndexRelationDescs`, calling `index_insert` for each
- `src/backend/access/index/indexam.c:index_insert` → invokes `aminsert` from the AM's `IndexAmRoutine`
- For btree: `aminsert = btinsert` (`src/backend/access/nbtree/nbtree.c`) → `_bt_doinsert` (`src/backend/access/nbtree/nbtinsert.c`) does the tree descent, finds the leaf, and inserts
- WAL record formats: `src/include/access/heapam_xlog.h`, `src/include/access/nbtxlog.h`

---

← Previous: [04b_cold_cache.md](04b_cold_cache.md) | Next: [06_page_extension.md](06_page_extension.md) →
