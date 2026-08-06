# 08. non-HOT UPDATE — change an indexed column

Thesis: §4.4.4
Prerequisite: state at the end of [07_hot_update.md](07_hot_update.md) (chain `lp=75 → lp=76` on page 73 for id=10100).

`zipcode` is indexed. Updating it forces `HeapDetermineColumnsInfo` to return "indexed column changed" → HOT not viable. `heap_update` writes a new tuple version, then `ExecInsertIndexTuples` walks every index and inserts a fresh entry pointing at the new TID. The OLD entries remain in the indexes, stale.

## Capture and inspect

```sql
SELECT pg_current_wal_lsn() AS lsn1 \gset

-- UPDATE on `zipcode` (IS indexed) → HOT not viable → ExecInsertIndexTuples
-- runs for each index, emitting INSERT_LEAF per index.
UPDATE person SET zipcode = '99999' WHERE id = 10100;

SELECT pg_current_wal_lsn() AS lsn2 \gset

SELECT start_lsn, resource_manager, record_type, record_length
FROM pg_get_wal_records_info(:'lsn1', :'lsn2')
ORDER BY start_lsn;
```

```
 start_lsn | resource_manager |  record_type   | record_length
-----------+------------------+----------------+---------------
 0/20353A8 | Heap             | UPDATE         |           115
 0/2035420 | Btree            | INSERT_LEAF    |            72
 0/2035470 | Btree            | INSERT_LEAF    |          8033
 0/2037498 | Transaction      | COMMIT         |            34
```

Now we see **`Heap / UPDATE`** (not `HOT_UPDATE`) plus **two `Btree / INSERT_LEAF`** records — one for each index. Both indexes get a fresh entry pointing at the new tuple version. The 8033-byte `INSERT_LEAF` on the second index is a full page image (first dirty of the leaf in the current checkpoint cycle).

## Heap state — three-link chain

```sql
SELECT lp, lp_flags, t_xmin, t_xmax, t_ctid, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('person', 73))
ORDER BY lp DESC LIMIT 4;
```

```
 lp | lp_flags | t_xmin | t_xmax | t_ctid  | t_infomask | t_infomask2
----+----------+--------+--------+---------+------------+-------------
 77 |        1 |    776 |      0 | (73,77) |      10498 |           4
 76 |        1 |    775 |    776 | (73,77) |       9474 |       32772
 75 |        1 |    774 |    775 | (73,76) |       1282 |       16388
```

A 3-link chain now lives on page 73: `lp=75 → lp=76 → lp=77`. Three versions of the same logical row.

Index state:
- `person_pkey` originally had one entry `id=10100 → (73,75)` (set when the row was first inserted at lp=75). After the HOT update in Demo 07, the index was untouched — still pointing at lp=75. After this non-HOT update, a **new** entry is added: `id=10100 → (73,77)`. The old `(73,75)` entry remains in the index, stale.
- `person_zipcode_idx` originally had `'10042' → (73,75)`. Same story: HOT update did not change it. Now a new entry `'99999' → (73,77)` is added. The `'10042'` entry remains.

Reader behaviour:
- A scan on `id=10100` via pkey gets *two* matching index entries → fetches `(73,75)` and `(73,77)`. The `(73,75)` entry resolves to no visible tuple (the chain ends at lp=76, which is dead), only `(73,77)` yields the live version — MVCC visibility eliminates the old version and the row is returned **once**.
- A scan on `zipcode='10042'` via zipcode_idx gets the old entry → fetches `(73,75)` → the HOT chain ends at lp=76, which is dead (`t_xmax=776`, committed) → **no visible tuple returned**. The stale entry returns no row but the executor pays for the heap fetch.

This is why long-running update workloads on indexed columns inflate index size and slow down lookups: index entries accumulate but cannot be removed until VACUUM, `kill_prior_tuple` opportunism, or bottom-up deletion reaches them.

The new tuple at `lp=77` does **not** carry `HEAP_ONLY_TUPLE` — it is a regular version that index entries point at directly.

## Source path

- `src/backend/executor/nodeModifyTable.c:ExecUpdate` → `heap_update`
- `HeapDetermineColumnsInfo` finds an indexed column changed → `use_hot_update = false`
- `heap_update` writes the new tuple, emits `XLOG_HEAP_UPDATE`
- `src/backend/executor/execIndexing.c:ExecInsertIndexTuples` walks `ResultRelInfo->ri_IndexRelationDescs`, calls `index_insert` for each → btree emits `INSERT_LEAF`
- Old index entries are left in place; cleanup deferred to `kill_prior_tuple` ([Demo 09b](09b_kill_prior_tuple.md)), `btbulkdelete` (Demo 11), or bottom-up deletion (Demo 12)

---

← Previous: [07_hot_update.md](07_hot_update.md) | Next: [09_delete.md](09_delete.md) →
