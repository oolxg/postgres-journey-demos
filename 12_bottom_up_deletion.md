# 12. Bottom-up deletion — opportunistic cleanup at INSERT time

Thesis: §4.5.2
Prerequisite: [00_setup.md](00_setup.md) on a fresh DB, server started with `autovacuum = off`. Captured on an unmodified PostgreSQL 18.1 build.

Stale index entries normally wait for VACUUM (Demo 11) or for a scan to set `LP_DEAD` via `kill_prior_tuple` (Demo 09b). PG14+ adds a third path: **bottom-up deletion**. When an insert finds its leaf full and simple `LP_DEAD` deletion did not free enough room, btree looks at groups of entries with the same key and asks the heap which of the row versions behind them are dead. Dead ones are removed in one batch, and the split is often avoided. Function: `nbtdedup.c:_bt_bottomupdel_pass`.

The pass runs only if `indexUnchanged || uniquedup` (`nbtinsert.c:2774`): the new entry comes from an UPDATE that did not change *this* index's key (hint computed per index at `execIndexing.c:443`), or the index is unique and already holds the key.

## Workload that isolates it on `person_zipcode_idx`

The UPDATE flips the sign of 100 primary keys. `id` changes, so no update can be HOT and every index gets a new entry. `zipcode` does not change, so every new `person_zipcode_idx` entry carries the `indexUnchanged` hint.

The other reclaim paths are switched off:

- `autovacuum = off` → `btbulkdelete` never runs.
- Sequential scan forced → no index scan, so `kill_prior_tuple` never sets `LP_DEAD`.
- `person_zipcode_idx` is not unique → no uniqueness check, which is the other place that sets `LP_DEAD` (`nbtinsert.c:685`, inside `_bt_check_unique`).

Each round commits. A version that the *running* transaction deleted is `HEAPTUPLE_DELETE_IN_PROGRESS`, not dead (`heapam_visibility.c:1270`), so bottom-up could not remove anything inside one transaction (see the last section).

```sql
SET enable_indexscan = off;
SET enable_bitmapscan = off;
EXPLAIN (COSTS OFF) UPDATE person SET id = -id WHERE id BETWEEN 200 AND 299 OR id BETWEEN -299 AND -200;
SELECT pg_current_wal_lsn() AS lsn_before \gset
DO $$
BEGIN
  FOR i IN 1..200 LOOP
    UPDATE person SET id = -id
     WHERE id BETWEEN 200 AND 299 OR id BETWEEN -299 AND -200;
    COMMIT;
  END LOOP;
END $$;
SELECT pg_current_wal_lsn() AS lsn_after \gset

-- WAL by record type
SELECT resource_manager, record_type, count(*) AS cnt
FROM pg_get_wal_records_info(:'lsn_before', :'lsn_after')
WHERE resource_manager IN ('Btree','Heap','Heap2')
GROUP BY 1, 2 ORDER BY 1, 2;

-- Btree records per index
SELECT c.relname, b.record_type, count(DISTINCT b.start_lsn) AS records
FROM pg_get_wal_block_info(:'lsn_before', :'lsn_after') b
JOIN pg_class c ON c.relfilenode = b.relfilenode
WHERE b.resource_manager = 'Btree'
GROUP BY 1, 2 ORDER BY 1, 2;

SELECT relname, pg_relation_size(oid)/8192 AS pages FROM pg_class
WHERE relname IN ('person','person_pkey','person_zipcode_idx') ORDER BY 1;
```

```
                                                QUERY PLAN
----------------------------------------------------------------------------------------------------------
 Update on person
   ->  Seq Scan on person
         Filter: (((id >= 200) AND (id <= 299)) OR ((id >= '-299'::integer) AND (id <= '-200'::integer)))
(3 rows)

 resource_manager |   record_type   |  cnt
------------------+-----------------+-------
 Btree            | DEDUP           |   152
 Btree            | DELETE          |   759
 Btree            | INSERT_LEAF     | 39993
 Btree            | INSERT_UPPER    |     7
 Btree            | SPLIT_L         |     4
 Btree            | SPLIT_R         |     3
 Heap             | LOCK            |  5325
 Heap             | UPDATE          | 19930
 Heap             | UPDATE+INIT     |    70
 Heap2            | PRUNE_ON_ACCESS |   317
(10 rows)

      relname       | record_type  | records
--------------------+--------------+---------
 person_pkey        | DELETE       |      90
 person_pkey        | INSERT_LEAF  |   19999
 person_pkey        | INSERT_UPPER |       1
 person_pkey        | SPLIT_L      |       1
 person_zipcode_idx | DEDUP        |     152
 person_zipcode_idx | DELETE       |     669
 person_zipcode_idx | INSERT_LEAF  |   19994
 person_zipcode_idx | INSERT_UPPER |       6
 person_zipcode_idx | SPLIT_L      |       3
 person_zipcode_idx | SPLIT_R      |       3
(10 rows)

      relname       | pages
--------------------+-------
 person             |   144
 person_pkey        |    31
 person_zipcode_idx |    17
(3 rows)
```

A second run from a fresh setup gave the same per-index counts.

No `LP_DEAD` item on the zipcode index after the run:

```sql
SELECT count(*) FILTER (WHERE dead) AS lp_dead_items, count(*) AS items
FROM generate_series(1, (pg_relation_size('person_zipcode_idx')/8192 - 1)::int) AS blk,
     LATERAL bt_page_items('person_zipcode_idx', blk);
```

```
 lp_dead_items | items
---------------+-------
             0 |  1796
(1 row)
```

### Attribution

`Btree / DELETE` has one emitting site, `XLogInsert(RM_BTREE_ID, XLOG_BTREE_DELETE)` in `_bt_delitems_delete` (`nbtpage.c:1369`), reached only through `_bt_delitems_delete_check`. That function has two callers: the simple `LP_DEAD` pass (`nbtinsert.c:2912`) and the bottom-up pass (`nbtdedup.c:410`). The zipcode index never held an `LP_DEAD` item, so its **669 DELETE records come from the bottom-up pass**. The 90 on `person_pkey` are not attributed: a unique index also gets `LP_DEAD` bits from its uniqueness checks, so the simple pass is possible there.

## Control: the same workload with an old snapshot held open

A second session keeps a `REPEATABLE READ` snapshot for the whole run, so none of the superseded versions is dead:

```sql
-- session 2, started before the workload and kept open until it ends
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM person;
SELECT pg_sleep(600);
```

Same setup and same workload in session 1:

```
 resource_manager | record_type  |  cnt
------------------+--------------+-------
 Btree            | DEDUP        |   877
 Btree            | INSERT_LEAF  | 39950
 Btree            | INSERT_UPPER |    50
 Btree            | SPLIT_L      |    26
 Btree            | SPLIT_R      |    24
 Heap             | LOCK         | 14744
 Heap             | UPDATE       | 19853
 Heap             | UPDATE+INIT  |   147
(8 rows)

      relname       | record_type  | records
--------------------+--------------+---------
 person_pkey        | DEDUP        |     424
 person_pkey        | INSERT_LEAF  |   19978
 person_pkey        | INSERT_UPPER |      22
 person_pkey        | SPLIT_L      |      14
 person_pkey        | SPLIT_R      |       8
 person_zipcode_idx | DEDUP        |     453
 person_zipcode_idx | INSERT_LEAF  |   19972
 person_zipcode_idx | INSERT_UPPER |      28
 person_zipcode_idx | SPLIT_L      |      12
 person_zipcode_idx | SPLIT_R      |      16
(10 rows)

      relname       | pages
--------------------+-------
 person             |   221
 person_pkey        |    52
 person_zipcode_idx |    39
(3 rows)

 lp_dead_items | items
---------------+-------
             0 |  2685
(1 row)
```

No `DELETE` record at all. The zipcode index splits 28 times instead of 6 and ends at 39 pages instead of 17 (it starts at 11). That difference is what bottom-up deletion saved.

## Why each round commits

The same number of updates inside one transaction, changing `zipcode` (the first version of this demo), with the default plan:

```sql
DO $$
DECLARE
  i int;
BEGIN
  FOR i IN 1..200 LOOP
    UPDATE person
       SET zipcode = ((10100 + (i * 31 + 17) % 1000))::text
     WHERE id BETWEEN 200 AND 299;
  END LOOP;
END $$;
```

```
 Update on person
   ->  Index Scan using person_pkey on person
         Index Cond: ((id >= 200) AND (id <= 299))

 resource_manager |   record_type   |  cnt
------------------+-----------------+-------
 Btree            | DEDUP           |   782
 Btree            | INSERT_LEAF     | 39953
 Btree            | INSERT_UPPER    |    47
 Btree            | SPLIT_L         |    21
 Btree            | SPLIT_R         |    26
 Heap             | LOCK            | 14744
 Heap             | UPDATE          | 19853
 Heap             | UPDATE+INIT     |   147
 Heap2            | PRUNE_ON_ACCESS |     1
(9 rows)
```

No `DELETE` record: every superseded version was deleted by the still-running transaction, so neither pass may remove it. Note also that the default plan is an Index Scan, and that both indexes receive an entry per update (≈ 40 000 `INSERT_LEAF`).

## How the heuristic chooses candidates

Simplified from `nbtree/nbtdedup.c:_bt_bottomupdel_pass`:

1. Walk the leaf, group entries by key.
2. Treat groups with several entries as likely version duplicates.
3. Sort candidate TIDs by heap block, hand them to `table_index_delete_tuples` (heap side: `heap_index_delete_tuples`, `heapam.c`).
4. The heap tests each TID against a non-vacuumable snapshot (`heapam.c:8135`) and visits at most `BOTTOMUP_MAX_NBLOCKS` = 6 heap blocks per pass (`heapam.c:184`).
5. Btree calls `_bt_delitems_delete` → one `XLOG_BTREE_DELETE` record per affected page.

## Source path

- `src/backend/access/nbtree/nbtinsert.c:_bt_delete_or_dedup_one_page` — dispatcher (order: simple deletion, bottom-up, dedup, then split)
- `src/backend/access/nbtree/nbtinsert.c:2774` — `indexUnchanged || uniquedup` condition
- `src/backend/executor/execIndexing.c:443` — per-index `indexUnchanged` hint
- `src/backend/access/nbtree/nbtinsert.c:685` — `_bt_check_unique` sets `LP_DEAD` (unique indexes only)
- `src/backend/access/nbtree/nbtdedup.c:_bt_bottomupdel_pass` — the bottom-up pass
- `src/backend/access/nbtree/nbtpage.c:_bt_delitems_delete` — removal + WAL emission

This is the third reclaim path, complementing `kill_prior_tuple` ([Demo 09b](09b_kill_prior_tuple.md)) and `btbulkdelete` (Demo 11).

---

← Previous: [11_vacuum.md](11_vacuum.md) | Next: [13_page_split.md](13_page_split.md) →
