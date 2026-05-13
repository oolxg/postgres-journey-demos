# 09. DELETE — heap-only modification, indexes untouched

Thesis: §4.3.5
Prerequisite: [00_setup.md](00_setup.md).

`heap_delete` sets `t_xmax` on the heap tuple, sets `HEAP_XMAX_EXCL_LOCK` infomask bits, emits `XLOG_HEAP_DELETE`, and stops. **No index callback is invoked.** Index entries remain in place — stale, pointing at heap tuples whose `t_xmax` is now set. Cleanup is deferred to one of three reclaim paths (Demo 11, Demo 12, or `kill_prior_tuple` opportunism during a future scan).

## Capture

```sql
-- Pre-DELETE index tuple counts. pgstattuple visits every tuple to compute these
-- (expensive on big tables; cheap here at 10k rows).
SELECT 'person'             AS rel, tuple_count, dead_tuple_count FROM pgstattuple('person')
UNION ALL
SELECT 'person_zipcode_idx', tuple_count, dead_tuple_count FROM pgstattuple('person_zipcode_idx');

SELECT pg_current_wal_lsn() AS lsn1 \gset

-- 10 rows on heap block 0 (id 100..109 are within the first ~135 rows).
DELETE FROM person WHERE id BETWEEN 100 AND 109;

SELECT pg_current_wal_lsn() AS lsn2 \gset

-- Aggregate WAL counts. Expect 10 Heap/DELETE records, ZERO Btree records.
SELECT resource_manager, record_type, count(*) AS cnt
FROM pg_get_wal_records_info(:'lsn1', :'lsn2')
GROUP BY 1, 2
ORDER BY 1, 2;
```

```
 resource_manager | record_type | cnt
------------------+-------------+-----
 Heap             | DELETE      |  10
 Transaction      | COMMIT      |   1
```

Zero Btree records. The index AM never sees the DELETE.

## Pre vs post — index counts unchanged

```sql
-- Same pgstattuple queries as above, after the DELETE.
SELECT 'person'             AS rel, tuple_count, dead_tuple_count FROM pgstattuple('person')
UNION ALL
SELECT 'person_zipcode_idx', tuple_count, dead_tuple_count FROM pgstattuple('person_zipcode_idx');
```

```
        rel         | tuple_count | dead_tuple_count
--------------------+-------------+------------------
 person             |        9990 |               10
 person_zipcode_idx |         334 |                0
```

Heap now has 10 dead tuples (`t_xmax` set, will be cleaned up by VACUUM). Index `tuple_count` is **identical** before and after — 334. The index's view of the deleted rows is unchanged. Each former-live row's index entry is now stale: it points at a heap tuple whose `t_xmax` is set to a committed deleter.

## How visibility works for SELECT after DELETE

1. The btree search still returns the matching index entries (e.g. `'10000' → (0,99)`).
2. For each TID, the executor calls `heap_fetch` to read the heap tuple.
3. `HeapTupleSatisfiesMVCC` sees `t_xmax` set and committed → tuple is dead from this snapshot's view → discarded.
4. Optionally, the AM marks the index line pointer `LP_DEAD` so a future inserter can reclaim that slot without VACUUM (`kill_prior_tuple`).

Confirm: SELECT via the just-deleted ids returns nothing, but the index still has entries:

```sql
SELECT id FROM person WHERE id BETWEEN 100 AND 109;   -- 0 rows

-- bt_page_items returns one row per index tuple on a given leaf.
-- The deleted ids' entries are still there until VACUUM (Demo 11) or
-- kill_prior_tuple sweeps them.
SELECT count(*) FROM bt_page_items('person_pkey', 1);
```

## Source path

- `src/backend/executor/nodeModifyTable.c:ExecDelete` → `heap_delete` (`src/backend/access/heap/heapam.c`)
- `heap_delete` sets `t_xmax`, sets the deletion infomask bits, and emits `XLOG_HEAP_DELETE`
- `ExecInsertIndexTuples` is NOT called for DELETEs; no `index_delete` exists in the heap_delete code path
- The index AM never sees the DELETE. Stale entries persist until reclaim (Demo 11, Demo 12, or `kill_prior_tuple`)

---

← Previous: [08_non_hot_update.md](08_non_hot_update.md) | Next: [09b_kill_prior_tuple.md](09b_kill_prior_tuple.md) →
