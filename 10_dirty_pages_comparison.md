# 10. Dirty-page signatures — HOT vs non-HOT vs DELETE

Thesis: §4.3.6
Prerequisite: [00_setup.md](00_setup.md), then `INSERT INTO person VALUES (10100, 'Bef', '10042', 30);` to set up the row we'll UPDATE.

The WAL trail in Demos 07–09 already shows which records each DML emits. A second angle on the same story: which buffer-pool pages each DML *dirties*. `CHECKPOINT` flushes all dirty pages to disk before each demo, so the dirty bits seen afterwards come exclusively from the operation that follows.

```sql
-- Need the row at id=10100 for the UPDATE phases.
INSERT INTO person VALUES (10100, 'Bef', '10042', 30);
```

## A) HOT UPDATE: heap only

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

HOT update dirties only the heap. Both indexes stay clean — `update_indexes=false` in `heap_update`'s return path skips `ExecInsertIndexTuples` entirely.

## B) non-HOT UPDATE: heap + one leaf per index

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

## C) DELETE: heap + VM, indexes untouched

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
| HOT UPDATE       | yes (1 page) | no  | no  | no  |
| non-HOT UPDATE   | yes (1 page) | no  | yes (1 leaf) | yes (1 leaf) |
| DELETE (10 rows) | yes (1 page) | yes | no  | no  |

The dirty-bit pattern is the operational signature of each DML class. Reading it from `pg_buffercache` is the buffer-manager-level mirror of the WAL trail: WAL says what was logged, dirty bits say what is currently in memory waiting to be flushed.

## Source path

- HOT UPDATE: `src/backend/access/heap/heapam.c:heap_update` (`use_hot_update=true`) — no index dispatch
- non-HOT: same `heap_update` + `ExecInsertIndexTuples` → one Btree dirty per index
- DELETE: `heap_delete` + `visibilitymap_clear` (`src/backend/access/heap/visibilitymap.c`) clears the `all_visible` bit → VM page becomes dirty

---

← Previous: [09b_kill_prior_tuple.md](09b_kill_prior_tuple.md) | Next: [11_vacuum.md](11_vacuum.md) →
