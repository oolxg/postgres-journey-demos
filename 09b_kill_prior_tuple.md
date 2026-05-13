# 09b. kill_prior_tuple — opportunistic LP_DEAD set by a reader

Thesis: §4.3.5
Prerequisite: [00_setup.md](00_setup.md). Run on a fresh DB before any other DELETE/UPDATE demos so dead-row bookkeeping is unambiguous.

DELETE (Demo 09) leaves index entries pointing at heap tuples whose `t_xmax` is now committed. The next scan that walks those entries does the rest of the work:

1. The btree returns each index entry's TID.
2. The executor calls `index_fetch_heap`, which reads the heap and runs the visibility check.
3. When the heap tuple is dead from *every* active snapshot's view, `index_fetch_heap` sets `scan->kill_prior_tuple = all_dead` (`indexam.c:699`).
4. On the next index call, the btree AM sees the flag and records the dead item in `so->killedItems[]` (`nbtree.c:251`).
5. Right before the scan moves off this leaf, `_bt_killitems` (`nbtutils.c:3265`) re-locks the buffer, walks `killedItems[]`, and sets the `LP_DEAD` bit on each matching item identifier.
6. The next inserter into this leaf finds the `LP_DEAD` bits during `_bt_simpledel_pass` and reclaims those slots without waiting for VACUUM.

The bit is a hint, not a commitment: it can be lost without harm (e.g. if the page LSN changed between scan and kill, `_bt_killitems` gives up entirely; see `nbtutils.c:3286`). What it cannot do is point at a live tuple — the visibility check that gates it is conservative.

## Capture

Output below is from this thesis's PG 18.1 build, captured live against the 10000-row `journey` database.

### Step 1 — identify the leaf

```sql
-- The leftmost pkey leaf is block 1 on a freshly-populated 10000-row table
-- (block 0 = metapage, block 3 = root). Confirm via bt_page_stats: type='l'
-- and btpo_level=0 mean a leaf page.
SELECT blkno, type, live_items, btpo_level
FROM bt_page_stats('person_pkey', 1);
```

```
 blkno | type | live_items | btpo_level
-------+------+------------+-----------
     1 |  l   |        367 |          0
```

### Step 2 — pre-DELETE baseline

```sql
-- No LP_DEAD bits anywhere on the leaf.
SELECT count(*) FILTER (WHERE dead) AS dead_items,
       count(*)                     AS total_items
FROM bt_page_items('person_pkey', 1);
```

```
 dead_items | total_items
------------+-------------
          0 |         367
```

### Step 3 — DELETE 10 rows

```sql
DELETE FROM person WHERE id BETWEEN 100 AND 109;
-- DELETE 10
```

### Step 4 — post-DELETE, pre-scan

```sql
-- DELETE does not touch the index, so the dead-bit count is unchanged.
SELECT count(*) FILTER (WHERE dead) AS dead_items
FROM bt_page_items('person_pkey', 1);
```

```
 dead_items
------------
          0
```

### Step 5 — scan that walks the stale entries

```sql
-- Returns 0 (rows are dead) but internally walks 10 index TIDs, calls
-- index_fetch_heap for each, gets all_dead=true ten times, and emits the
-- kill_prior_tuple flag. _bt_killitems writes the LP_DEAD bits before
-- the scan releases the leaf.
SELECT count(*) AS rows_returned FROM person WHERE id BETWEEN 100 AND 109;
```

```
 rows_returned
---------------
             0
```

### Step 6 — post-scan dead-bit count

```sql
SELECT count(*) FILTER (WHERE dead) AS dead_items,
       count(*)                     AS total_items
FROM bt_page_items('person_pkey', 1);
```

```
 dead_items | total_items
------------+-------------
         10 |         367
```

Ten LP_DEAD bits, exactly matching the ten rows the scan touched.

### Step 7 — inspect the ten dead entries

```sql
-- WHERE on a function-scan column raises "could not open relation with OID 0"
-- on this build's instrumented planner. Workaround: materialise into a temp
-- table, then filter. Drop the temp table at the end for re-runnability.
CREATE TEMP TABLE leaf_snap AS
  SELECT itemoffset, dead, htid FROM bt_page_items('person_pkey', 1);
SELECT itemoffset, dead, htid FROM leaf_snap WHERE dead ORDER BY itemoffset;
DROP TABLE leaf_snap;
```

```
 itemoffset | dead |  htid
------------+------+---------
        101 |  t   | (0,100)
        102 |  t   | (0,101)
        103 |  t   | (0,102)
        104 |  t   | (0,103)
        105 |  t   | (0,104)
        106 |  t   | (0,105)
        107 |  t   | (0,106)
        108 |  t   | (0,107)
        109 |  t   | (0,108)
        110 |  t   | (0,109)
```

The `htid` column is the heap TID each index entry points at — exactly the ten rows just deleted by `id BETWEEN 100 AND 109`. The btree `itemoffset` numbering starts at 1, so id=100 sits at offset 101 (offset 1 is reserved for the leaf's `high_key` when one exists; on the leftmost leaf with no left sibling, the first data offset is 2 — see `nbtree.h:P_FIRSTDATAKEY` and the layout described in Demo 00).

## Why this is opportunistic, not eager

A few conditions must hold for the bit to make it onto the page:

- The deleting transaction must be committed AND below the global xmin horizon. If a concurrent snapshot can still see the row, `index_fetch_heap` reports `all_dead = false` and the flag stays clear.
- The scan must reach `_bt_killitems`, which only runs when the AM moves off the leaf (next page or scan end). A scan that finds its target on the first item and stops without advancing may skip the kill phase.
- For scans that drop the buffer pin eagerly (`so->dropPin`), the page LSN must not have changed since `_bt_readpage`. Concurrent splits or vacuum on this leaf cancel the kill (`nbtutils.c:3286`: "we totally give up on setting LP_DEAD bits when the page LSN changed").

If any condition fails, the dead bit is simply not set — VACUUM or bottom-up deletion will catch the entry later. The mechanism is best-effort.

## What the bit enables next

`_bt_simpledel_pass` (`nbtinsert.c`, dispatched from `_bt_delete_or_dedup_one_page` at `nbtinsert.c:2683`) walks the leaf, collects all entries with `LP_DEAD` set, and removes them in one batch when a new entry needs space. The next inserter into block 1, once block 1 fills up, will reclaim the ten slots we just marked — no VACUUM run required. Demos 11 (full VACUUM) and 12 (bottom-up) cover the other two reclaim paths; this demo covers the cheapest one.

## Source path

- `src/backend/access/index/indexam.c:699` — `scan->kill_prior_tuple = all_dead;` set inside `index_fetch_heap` after the visibility check
- `src/backend/access/nbtree/nbtree.c:251` — the btree AM checks the flag on the next call and accumulates the killed item index in `so->killedItems[]`
- `src/backend/access/nbtree/nbtutils.c:3265` (`_bt_killitems`) — re-locks the leaf buffer, walks `killedItems[]`, sets `LP_DEAD` on each matching itemid, then sets `BTP_HAS_GARBAGE` on the page
- `src/backend/access/nbtree/nbtutils.c:3286` — LSN-stability check that gates the kill for `dropPin` scans
- `src/backend/access/nbtree/nbtinsert.c:_bt_simpledel_pass` (dispatched from `_bt_delete_or_dedup_one_page` at `nbtinsert.c:2683`) — consumes the LP_DEAD bits at insert time

(Block 1 above is the leftmost pkey leaf on a freshly-populated 10000-row table; if your schema differs, walk `bt_page_stats` from the root and pick the leaf that owns ids 100..109.)

---

← Previous: [09_delete.md](09_delete.md) | Next: [10_dirty_pages_comparison.md](10_dirty_pages_comparison.md) →
