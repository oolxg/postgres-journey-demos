# 10. Dirty-page signatures — INSERT vs HOT vs non-HOT vs DELETE

Thesis: §4.4.6
Prerequisite: [00_setup.md](00_setup.md), then `VACUUM person;` so every heap page starts out all-visible. Phase A inserts the row the UPDATE phases reuse.

The WAL trail in Demos 07–09 already shows which records each DML emits. A second angle on the same story: which buffer-pool pages each DML *dirties*. `CHECKPOINT` flushes all dirty pages to disk before each demo, so the dirty bits seen afterwards come exclusively from the operation that follows.

## A) INSERT: heap + VM + one leaf per index

The widest signature of the four. The row needs a heap page, each index needs a new
entry, and the heap page was all-visible until now, so its visibility-map bit has to go.

```sql
-- Confirm the starting point: VACUUM left every page all-visible.
SELECT count(*) FILTER (WHERE all_visible) AS all_visible_pages, count(*) AS total
FROM pg_visibility_map('person');
--  all_visible_pages | total
-- -------------------+-------
--                 74 |    74

CHECKPOINT;

-- One row. It lands on page 73, the last heap page, which the initial load left partly empty.
INSERT INTO person VALUES (10100, 'Bef', '10042', 30);

SELECT c.relname,
       b.relforknumber AS fork,    -- fork=0 main heap, fork=2 visibility map
       b.relblocknumber AS blk,
       b.usagecount AS uc
FROM pg_buffercache b
JOIN pg_class c ON c.relfilenode = b.relfilenode
WHERE c.relname IN ('person','person_pkey','person_zipcode_idx')
  AND b.isdirty = TRUE
ORDER BY c.relname, b.relforknumber, b.relblocknumber;
```

```
      relname       | fork | blk | uc
--------------------+------+-----+----
 person             |    0 |  73 |  5     -- heap data: the new row
 person             |    2 |   0 |  5     -- VM page: all_visible bit cleared for page 73
 person_pkey        |    0 |  29 |  5     -- leaf taking id=10100
 person_zipcode_idx |    0 |   9 |  5     -- leaf taking zipcode='10042'
```

All four kinds of file at once. The free-space map (fork 1) stays clean, because the page
still had room and no extension was needed (Demo 06 covers the case where it does not).

The VM row is conditional, and repeating the INSERT proves it:

```sql
CHECKPOINT;
INSERT INTO person VALUES (10101, 'Bef2', '10042', 30);
-- same query as above
```

```
      relname       | fork | blk
--------------------+------+-----
 person             |    0 |  73
 person_pkey        |    0 |  29
 person_zipcode_idx |    0 |   9
```

No VM row this time. `visibilitymap_clear` only dirties the map page when the bit was
actually set, and the first INSERT already cleared it. The same reasoning is why phase C
below shows no VM page: by then page 73 has been written to several times.

## B) HOT UPDATE: heap only

```sql
-- CHECKPOINT flushes every dirty page to disk and marks them clean in shared
-- buffers. Subsequent dirty marks come strictly from the next operation.
CHECKPOINT;

-- name not indexed → HOT viable → indexes untouched.
UPDATE person SET name = 'HOTAfter' WHERE id = 10100;

-- Filter to rows with isdirty=true for our three relations. Join via pg_class
-- to avoid the WHERE-on-function-scan-column issue.
SELECT c.relname,
       b.relforknumber AS fork,
       b.relblocknumber AS blk,
       b.usagecount AS uc
FROM pg_buffercache b
JOIN pg_class c ON c.relfilenode = b.relfilenode
WHERE c.relname IN ('person','person_pkey','person_zipcode_idx')
  AND b.isdirty = TRUE
ORDER BY c.relname, b.relforknumber, b.relblocknumber;
```

```
 relname | fork | blk | uc
---------+------+-----+----
 person  |    0 |  73 |  5     -- heap page holding the row
```

HOT update dirties only the heap. Both indexes stay clean — `heap_update` sets `use_hot_update`, returning `*update_indexes = TU_None`, so `ExecUpdate`'s `updateIndexes != TU_None` guard skips `ExecInsertIndexTuples` entirely.

## C) non-HOT UPDATE: heap + one leaf per index

```sql
CHECKPOINT;

-- zipcode IS indexed → HOT not viable. heap_update writes new tuple;
-- ExecInsertIndexTuples inserts a fresh entry into both indexes.
UPDATE person SET zipcode = '99999' WHERE id = 10100;

SELECT c.relname,
       b.relforknumber AS fork,
       b.relblocknumber AS blk,
       b.usagecount AS uc
FROM pg_buffercache b
JOIN pg_class c ON c.relfilenode = b.relfilenode
WHERE c.relname IN ('person','person_pkey','person_zipcode_idx')
  AND b.isdirty = TRUE
ORDER BY c.relname, b.relforknumber, b.relblocknumber;
```

```
 relname            | fork | blk | uc
--------------------+------+-----+----
 person             |    0 |  73 |  5     -- new tuple version on same page
 person_pkey        |    0 |  29 |  5     -- pkey leaf: new entry id=10100 → (73,77)
 person_zipcode_idx |    0 |   2 |  5     -- zipcode_idx leaf: new entry '99999' → (73,77)
```

Three dirty pages — heap plus one leaf per index. Every index gets a fresh entry on a non-HOT update; the buffer-pool dirty bits are the in-memory evidence.

## D) DELETE: heap + VM, indexes untouched

```sql
CHECKPOINT;

-- 10 heap tuples on block 0 get t_xmax stamped. The page's all_visible bit
-- must be cleared so future readers know to re-check visibility → VM dirties.
DELETE FROM person WHERE id BETWEEN 100 AND 109;

SELECT c.relname,
       b.relforknumber AS fork,    -- fork=0 main heap, fork=2 visibility map
       b.relblocknumber AS blk,
       b.usagecount AS uc
FROM pg_buffercache b
JOIN pg_class c ON c.relfilenode = b.relfilenode
WHERE c.relname IN ('person','person_pkey','person_zipcode_idx')
  AND b.isdirty = TRUE
ORDER BY c.relname, b.relforknumber, b.relblocknumber;
```

```
 relname | fork | blk | uc
---------+------+-----+----
 person  |    0 |   0 |  5     -- heap data: t_xmax stamped on 10 tuples
 person  |    2 |   0 |  5     -- VM page: all_visible bit cleared
```

DELETE dirties the heap data page (`t_xmax` set per tuple) and the visibility-map page that covers it (the all-visible bit must be cleared so future scans know to re-check visibility). Indexes stay clean for the same reason WAL emits no `Btree` records: `heap_delete` never calls into the index AM.

## Summary

| Operation | heap dirty | VM dirty | pkey dirty | zipcode_idx dirty |
|---|---|---|---|---|
| INSERT (1 row)   | yes (1 page) | yes | yes (1 leaf) | yes (1 leaf) |
| HOT UPDATE       | yes (1 page) | no  | no  | no  |
| non-HOT UPDATE   | yes (1 page) | no  | yes (1 leaf) | yes (1 leaf) |
| DELETE (10 rows) | yes (1 page) | yes | no  | no  |

The dirty-bit pattern is the operational signature of each DML class. Reading it from `pg_buffercache` is the buffer-manager-level mirror of the WAL trail: WAL says what was logged, dirty bits say what is currently in memory waiting to be flushed.

## Source path

- INSERT: `heapam.c:heap_insert` + `ExecInsertIndexTuples` + `visibilitymap_clear` when the target page was all-visible
- HOT UPDATE: `src/backend/access/heap/heapam.c:heap_update` (`use_hot_update=true`) — no index dispatch
- non-HOT: same `heap_update` + `ExecInsertIndexTuples` → one Btree dirty per index
- DELETE: `heap_delete` + `visibilitymap_clear` (`src/backend/access/heap/visibilitymap.c`) clears the `all_visible` bit → VM page becomes dirty

---

← Previous: [09b_kill_prior_tuple.md](09b_kill_prior_tuple.md) | Next: [11_vacuum.md](11_vacuum.md) →
