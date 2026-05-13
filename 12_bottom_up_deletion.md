# 12. Bottom-up deletion — opportunistic cleanup at INSERT time

Thesis: §4.4.2
Prerequisite: [00_setup.md](00_setup.md). Run on a fresh DB (no prior UPDATEs/DELETEs) for the cleanest result.

Stale index entries normally wait for VACUUM (Demo 11) or for a scan to set `LP_DEAD` via `kill_prior_tuple`. PG14+ adds a third path: **bottom-up deletion**, triggered from `_bt_findinsertloc` when a leaf is full. Before splitting the leaf, btree looks at duplicate-key clusters on the leaf and asks the heap whether the corresponding tuples are still alive. Dead ones are removed in one batch; the leaf gets enough room, the split is avoided. Function: `nbtdedup.c:_bt_bottomupdel_pass`.

## Workload that forces it

Heavy non-HOT UPDATEs on an indexed column **without scans** (so `kill_prior_tuple` cannot fire) and **without VACUUM** (so `btbulkdelete` cannot run). Any `Btree / DELETE` WAL records produced under those conditions can only come from `_bt_bottomupdel_pass`.

```sql
SELECT pg_current_wal_lsn() AS lsn_before \gset

-- 200 iterations × 100 rows = 20 000 non-HOT UPDATEs on the indexed column.
-- Each iteration writes a new heap version and inserts a new zipcode_idx entry
-- (id is unchanged so pkey is NOT updated this round). Old versions become
-- dead immediately; their index entries pile up until a leaf fills.
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

SELECT pg_current_wal_lsn() AS lsn_after \gset

-- Aggregate WAL by (resource_manager, record_type).
SELECT resource_manager, record_type, count(*) AS cnt
FROM pg_get_wal_records_info(:'lsn_before', :'lsn_after')
WHERE resource_manager IN ('Btree','Heap')
GROUP BY 1, 2
ORDER BY 1, 2;
```

```
 resource_manager | record_type    | cnt
------------------+----------------+--------
 Btree            | DELETE         |    127     <- bottom-up deletions
 Btree            | INSERT_LEAF    |  20000
 Heap             | UPDATE         |  20000
```

127 `Btree / DELETE` records were emitted across the run. **No scans ran**, so `kill_prior_tuple` never set any `LP_DEAD` bit. **No VACUUM ran**, so `btbulkdelete` never executed. The only remaining producer is `_bt_bottomupdel_pass`, invoked from the leaf-full path during INSERT.

Without bottom-up deletion the index would have grown by 20 000 entries (one per UPDATE) plus enough splits to absorb the inflow. With bottom-up, batches of dead entries were reclaimed at insert time, deferring or avoiding splits.

## How the heuristic chooses candidates

Simplified from `nbtree/nbtdedup.c:_bt_bottomupdel_pass`:

1. Walk the leaf, group entries by leading key bytes.
2. For each group with multiple entries, treat the group as a candidate "version-chain duplicate" set.
3. Sort candidate TIDs by heap block, hand them to the heap callback `table_index_delete_tuples`.
4. Heap inspects each TID: if the tuple is dead from every snapshot, mark it for deletion.
5. Btree calls `_bt_delitems_delete` → emits one `XLOG_BTREE_DELETE` record per affected page.

## Source path

- `src/backend/access/nbtree/nbtinsert.c:_bt_findinsertloc` — leaf-full check
- `src/backend/access/nbtree/nbtinsert.c:_bt_delete_or_dedup_one_page` — dispatcher
- `src/backend/access/nbtree/nbtinsert.c:_bt_simpledel_pass` — first try, uses already-set `LP_DEAD` bits
- `src/backend/access/nbtree/nbtdedup.c:_bt_bottomupdel_pass` — second try, heuristic on posting/version groups
- `src/backend/access/nbtree/nbtpage.c:_bt_delitems_delete` — actual removal + WAL emission

This is the third reclaim path, complementing `kill_prior_tuple` (Demo 09) and `btbulkdelete` (Demo 11). The first two cover most workloads. Bottom-up deletion targets the specific case of UPDATE-heavy workloads where neither scans nor VACUUM run often enough to keep up.

---

← Previous: [11_vacuum.md](11_vacuum.md) | Next: [13_page_split.md](13_page_split.md) →
