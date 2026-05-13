# 07. HOT UPDATE — change a non-indexed column

Thesis: §4.3.3
Prerequisite: [00_setup.md](00_setup.md), then insert the row this demo updates: `INSERT INTO person VALUES (10100, 'Bef', '10042', 30);` (puts the row on page 73 with spare space).

`name` is not indexed; `id` and `zipcode` are. Updating only `name` keeps every indexed column unchanged → HOT-eligible. `heap_update` writes the new tuple on the **same page** and sets a forward link from the old tuple to the new one via `t_ctid`. Both indexes are skipped entirely: zero Btree WAL records.

## Capture and inspect

```sql
-- Insert the row this demo updates. New tuple lands on last page (page 73)
-- because earlier pages are full from populate; page 73 still has room.
INSERT INTO person VALUES (10100, 'Bef', '10042', 30);

-- Capture WAL LSN before / after; \gset saves the result into a psql variable.
SELECT pg_current_wal_lsn() AS lsn1 \gset

-- UPDATE only `name` (NOT in any index) → HOT-eligible.
UPDATE person SET name = 'After' WHERE id = 10100;

SELECT pg_current_wal_lsn() AS lsn2 \gset

-- pg_get_wal_records_info (pg_walinspect) returns one row per WAL record
-- between two LSNs. Used as the primary forensic tool for these demos.
SELECT start_lsn, resource_manager, record_type, record_length
FROM pg_get_wal_records_info(:'lsn1', :'lsn2')
ORDER BY start_lsn;
```

```
 start_lsn | resource_manager |   record_type    | record_length
-----------+------------------+------------------+---------------
 0/2034F70 | Heap             | HOT_UPDATE       |           102
 0/2034FD8 | Transaction      | COMMIT           |            34
```

One Heap record (`HOT_UPDATE`), then COMMIT. **Zero Btree records.** The promise of HOT: indexes are not touched.

## Heap state — the chain

```sql
-- Decode page 73 row-by-row. t_infomask / t_infomask2 carry the HOT flags
-- (HEAP_HOT_UPDATED, HEAP_ONLY_TUPLE) referenced in the prose below.
SELECT lp, lp_flags, lp_len, t_xmin, t_xmax, t_ctid, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('person', 73))
ORDER BY lp DESC LIMIT 4;
```

```
 lp | lp_flags | lp_len | t_xmin | t_xmax | t_ctid  | t_infomask | t_infomask2
----+----------+--------+--------+--------+---------+------------+-------------
 76 |        1 |     50 |    775 |      0 | (73,76) |      10498 |       32772
 75 |        1 |     52 |    774 |    775 | (73,76) |       1282 |       16388
 74 |        1 |     54 |    773 |      0 | (73,74) |      10498 |           4
 73 |        1 |     50 |    772 |      0 | (73,73) |       2050 |           4
```

Decoding the HOT chain (flags in `src/include/access/htup_details.h`):

- `lp=75` is the original tuple. `t_xmin=774` (insert), `t_xmax=775` (HOT-deleted). `t_ctid=(73,76)` — points forward to the new version. `t_infomask2` has `HEAP_HOT_UPDATED` set. Both index entries (pkey, zipcode_idx) still point at `(73,75)`.
- `lp=76` is the new tuple. `t_xmin=775` (the UPDATE's xid), `t_xmax=0` (live). `t_infomask2` has `HEAP_ONLY_TUPLE` set — **no index entry points to it directly**. Readers find it only by following the chain from `(73,75)` to `(73,76)`.

The same situation drawn against the heap page:

```
Index pkey                   Heap page 73
─────────────                ─────────────
                             ┌─────────────────────────────────────────┐
                             │ lp=73   xmin=772, xmax=0     ...        │ live (other row)
                             ├─────────────────────────────────────────┤
                             │ lp=74   xmin=773, xmax=0     ...        │ live (other row)
                             ├─────────────────────────────────────────┤
   id=10100 ──────────────▶  │ lp=75   xmin=774, xmax=775,  name='Bef' │ HEAP_HOT_UPDATED
                             │         t_ctid = (73,76)  ──┐           │
                             ├─────────────────────────────│───────────┤
                             │ lp=76   xmin=775, xmax=0,   ▼ name='Aft'│ HEAP_ONLY_TUPLE
                             │         t_ctid = (73,76) (self, current)│ (no index entry → here)
                             └─────────────────────────────────────────┘
```

- The index entry for `id=10100` was **not updated**. It still points at `(73,75)` — the original heap location.
- The reader arrives at `lp=75`, sees `xmax=775` (committed deleter), follows `t_ctid` to `lp=76`, reads the current version.
- `lp=76` is `HEAP_ONLY_TUPLE` — visible only via the chain. No index entry refers to it directly.
- Both indexes were spared an update: zero `Btree / INSERT_LEAF` WAL records for this UPDATE.

## Source path

- `src/backend/executor/nodeModifyTable.c:ExecUpdate` → `heap_update` (`src/backend/access/heap/heapam.c`)
- `heap_update` calls `HeapDetermineColumnsInfo` to compute which indexed columns changed → `use_hot_update` flag
- If HOT viable AND new tuple fits same page: `heap_update` writes the new tuple, sets `HEAP_HOT_UPDATED` on old, `HEAP_ONLY_TUPLE` on new, sets `t_ctid` chain. Emits `XLOG_HEAP_HOT_UPDATE`.
- Index update path is **skipped** entirely — `ExecInsertIndexTuples` is bypassed when `update_indexes` flag is false.

---

← Previous: [06_page_extension.md](06_page_extension.md) | Next: [08_non_hot_update.md](08_non_hot_update.md) →
