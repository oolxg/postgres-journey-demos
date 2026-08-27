# 15. Controlling btree deduplication — incremental vs rebuild vs off

Thesis: §4.4.8
Prerequisite: [00_setup.md](00_setup.md). Run the steps below in order — each one observes the state the previous one left.

Demo 00 shows that `person_zipcode_idx` is 11 pages while `person_pkey` is 30, and attributes the difference to deduplication. This demo answers the three follow-up questions that raises:

1. How are duplicate entries actually merged, and when?
2. Can the index be built fully merged from the start?
3. What can influence the merge — does VACUUM help, and is there a knob?

Short answers: an incremental insert workload merges lazily and leaves stragglers behind, a rebuild merges everything, VACUUM does nothing for deduplication, and the storage parameter `deduplicate_items` turns the feature off.

## State A: what an incremental insert workload leaves

This is the state Demo 00 ends in — the index was created empty and 10000 rows were inserted afterwards.

```sql
SELECT g AS blk, s.type, s.live_items, s.avg_item_size, s.free_size
FROM generate_series(1, 10) AS g,
     LATERAL bt_page_stats('person_zipcode_idx', g) s
ORDER BY g;
```

```
 blk | type | live_items | avg_item_size | free_size 
-----+------+------------+---------------+-----------
   1 | l    |         31 |           226 |       992
   2 | l    |         88 |            86 |       228
   3 | r    |          9 |            15 |      7976
   4 | l    |         30 |           234 |      1004
   5 | l    |         30 |           234 |      1004
   6 | l    |         44 |           162 |       836
   7 | l    |         16 |           466 |       628
   8 | l    |         33 |           213 |       968
   9 | l    |         33 |           213 |       968
  10 | l    |         33 |           213 |       968
```

Page 3 is the root, the other nine are leaves. The leaves are uneven: 16 items on one, 88 on another, and `avg_item_size` swings between 86 and 466 bytes. That unevenness *is* the lazy merge. `_bt_dedup_pass` runs only when a leaf is about to split, so each leaf carries whatever mixture of merged and unmerged entries its own split history produced.

Counting the whole index:

```sql
SELECT count(*)                                     AS leaf_items,
       count(*) FILTER (WHERE i.tids IS NOT NULL)   AS posting_tuples,
       count(*) FILTER (WHERE i.tids IS NULL)       AS plain_items,
       sum(coalesce(array_length(i.tids,1),1))      AS tids_total
FROM (VALUES (1),(2),(4),(5),(6),(7),(8),(9),(10)) AS v(blk),
     LATERAL bt_page_items('person_zipcode_idx', v.blk) i;
```

```
 leaf_items | posting_tuples | plain_items | tids_total 
------------+----------------+-------------+------------
        338 |            100 |         238 |      10008
```

The `tids` column is non-null only for posting tuples, which makes the split visible: 100 posting tuples, one per distinct zipcode, plus 238 plain entries. `tids_total` is 10008, which is the 10000 rows plus the 8 high keys of the eight non-rightmost leaves.

So 230 real entries never got merged. They are the stragglers — rows that arrived after the last dedup pass on their leaf and have not yet triggered another one.

## VACUUM does not deduplicate

The obvious guess is that VACUUM tidies this up. It does not.

```sql
VACUUM person;

SELECT count(*)                                     AS leaf_items,
       count(*) FILTER (WHERE i.tids IS NOT NULL)   AS posting_tuples,
       count(*) FILTER (WHERE i.tids IS NULL)       AS plain_items
FROM (VALUES (1),(2),(4),(5),(6),(7),(8),(9),(10)) AS v(blk),
     LATERAL bt_page_items('person_zipcode_idx', v.blk) i;
```

```
 leaf_items | posting_tuples | plain_items 
------------+----------------+-------------
        338 |            100 |         238
```

Identical to State A. VACUUM's index pass is `btbulkdelete`, which removes entries whose TIDs are dead. Nothing in that path merges live entries, so a table with no dead tuples gives VACUUM no work and the item counts do not move.

## REINDEX merges everything

A rebuild reads the whole index input in sorted order, which is the one situation where every duplicate group is visible at once.

```sql
REINDEX INDEX person_zipcode_idx;

SELECT g AS blk, s.type, s.live_items, s.avg_item_size, s.free_size
FROM generate_series(1, 10) AS g,
     LATERAL bt_page_stats('person_zipcode_idx', g) s
ORDER BY g;
```

```
 blk | type | live_items | avg_item_size | free_size 
-----+------+------------+---------------+-----------
   1 | l    |         13 |           569 |       688
   2 | l    |         13 |           569 |       688
   3 | r    |          9 |            15 |      7976
   4 | l    |         13 |           569 |       688
   5 | l    |         13 |           569 |       688
   6 | l    |         13 |           569 |       688
   7 | l    |         13 |           569 |       688
   8 | l    |         13 |           569 |       688
   9 | l    |         13 |           569 |       688
  10 | l    |          4 |           616 |      5668
```

Every leaf now holds exactly 13 items of the same average size, and the leftover goes to the rightmost leaf. The counts confirm what happened:

```sql
SELECT count(*)                                     AS leaf_items,
       count(*) FILTER (WHERE i.tids IS NOT NULL)   AS posting_tuples,
       count(*) FILTER (WHERE i.tids IS NULL)       AS plain_items,
       sum(coalesce(array_length(i.tids,1),1))      AS tids_total
FROM (VALUES (1),(2),(4),(5),(6),(7),(8),(9),(10)) AS v(blk),
     LATERAL bt_page_items('person_zipcode_idx', v.blk) i;
```

```
 leaf_items | posting_tuples | plain_items | tids_total 
------------+----------------+-------------+------------
        108 |            100 |           8 |      10008
```

338 items became 108. The posting tuple count is unchanged at 100, but the plain entries fell from 238 to 8 — and 8 is exactly the number of high keys. **Every single data entry now lives inside a posting tuple.** That is the answer to question 2: yes, a freshly built index is maximally deduplicated, and `REINDEX` (or `CREATE INDEX`) is how you get there.

The leaf now reads as one posting tuple per zipcode, in key order:

```sql
SELECT itemoffset, ctid, itemlen, substring(data for 20) AS data_first20
FROM bt_page_items('person_zipcode_idx', 1) LIMIT 5;
```

```
 itemoffset |   ctid    | itemlen |     data_first20     
------------+-----------+---------+----------------------
          1 | (16,1)    |      16 | 0d 31 30 30 31 32 00
          2 | (16,8292) |     616 | 0d 31 30 30 30 30 00
          3 | (16,8292) |     616 | 0d 31 30 30 30 31 00
          4 | (16,8292) |     616 | 0d 31 30 30 30 32 00
          5 | (16,8292) |     616 | 0d 31 30 30 30 33 00
```

Compare with the same query in Demo 00, where items 1, 3 and 4 were 16-byte singletons wedged between the posting tuples. Item 1 here is the page's high key (page 1 is not the rightmost leaf). Items 2 onward are consecutive zipcodes `'10000'`, `'10001'`, `'10002'`, `'10003'` — the `0d` byte is the varlena header of a five-character `text` value and the rest is ASCII.

Note what did **not** change: the index is still 11 pages. The consolidation removed per-entry overhead, not TID payload. 10000 TIDs at 6 bytes each is about 60 KB whatever the packing, so the file size is already near its floor in both states. The gain from a rebuild here is a cleaner leaf structure and free space at the end, not a smaller file.

## Turning deduplication off

The feature is a per-index storage parameter, on by default.

```sql
CREATE INDEX person_zipcode_nodedup_idx ON person (zipcode)
    WITH (deduplicate_items = off);

SELECT g AS blk, s.type, s.live_items, s.avg_item_size
FROM generate_series(1, 5) AS g,
     LATERAL bt_page_stats('person_zipcode_nodedup_idx', g) s
ORDER BY g;
```

```
 blk | type | live_items | avg_item_size 
-----+------+------------+---------------
   1 | l    |        367 |            16
   2 | l    |        367 |            16
   3 | r    |         28 |            23
   4 | l    |        367 |            16
   5 | l    |        367 |            16
```

Every leaf is now 367 uniform 16-byte entries, one per row.

```sql
SELECT count(*)                                     AS leaf_items,
       count(*) FILTER (WHERE i.tids IS NOT NULL)   AS posting_tuples,
       count(*) FILTER (WHERE i.tids IS NULL)       AS plain_items,
       sum(coalesce(array_length(i.tids,1),1))      AS tids_total
FROM (VALUES (1),(2),(4),(5),(6),(7),(8),(9),(10),(11),(12),(13),(14),(15),
             (16),(17),(18),(19),(20),(21),(22),(23),(24),(25),(26),(27),(28),(29)) AS v(blk),
     LATERAL bt_page_items('person_zipcode_nodedup_idx', v.blk) i;
```

```
 leaf_items | posting_tuples | plain_items | tids_total 
------------+----------------+-------------+------------
      10027 |              0 |       10027 |      10027
```

Zero posting tuples. 10027 is 10000 entries plus the 27 high keys of the 28 leaves.

## Summary

```sql
SELECT relname,
       coalesce(reloptions::text, '(default)')  AS options,
       pg_relation_size(oid) / 8192             AS pages,
       pg_size_pretty(pg_relation_size(oid))    AS size
FROM pg_class
WHERE relname IN ('person_pkey', 'person_zipcode_idx', 'person_zipcode_nodedup_idx')
ORDER BY relname;
```

```
          relname           |         options         | pages |  size  
----------------------------+-------------------------+-------+--------
 person_pkey                | (default)               |    30 | 240 kB
 person_zipcode_idx         | (default)               |    11 | 88 kB
 person_zipcode_nodedup_idx | {deduplicate_items=off} |    30 | 240 kB
```

The same 10000 TIDs cost 30 pages without deduplication and 11 with it. The undeduplicated zipcode index lands on exactly the pkey's 30 pages, which is the cleanest statement of what the feature buys: the pkey is that size because its keys are unique and there is nothing to merge, and the zipcode index would be that size too if merging were switched off.

| state | how it was built | pages | leaf items | posting tuples | plain entries |
|-------|------------------|-------|-----------|----------------|---------------|
| A incremental | index created empty, then 10000 inserts | 11 | 338 | 100 | 238 (8 are high keys) |
| B rebuilt | `REINDEX` | 11 | 108 | 100 | 8 (all high keys) |
| C off | `deduplicate_items = off` | 30 | 10027 | 0 | 10027 (27 are high keys) |

So the levers are: `REINDEX` to consolidate an index that inserts have fragmented, and `deduplicate_items` to switch merging off for an index where it does not pay. VACUUM is not a lever.

## Cleanup

The rest of the kit assumes the Demo 00 state, so drop the extra index and re-run `00_setup.md` before continuing:

```sql
DROP INDEX person_zipcode_nodedup_idx;
```

## Reproducibility note

The item and page counts above are deterministic for this workload — the sequence was run twice from a fresh `00_setup.md` and produced identical numbers, including the uneven per-leaf shape of State A.

One quirk of the instrumented build (see README): combining `bt_page_stats` and `bt_page_items` in a single query, or putting a `WHERE` filter on the `generate_series` column that feeds `LATERAL bt_page_items`, fails with `ERROR: could not open relation with OID 0`. The queries above avoid both shapes by listing the leaf blocks explicitly.

## Source paths

- Incremental merge: `src/backend/access/nbtree/nbtdedup.c:58` — `_bt_dedup_pass`
  - called from `src/backend/access/nbtree/nbtinsert.c:2780`, inside `_bt_delete_or_dedup_one_page`, which runs only when a leaf cannot fit a new entry (see Demo 12 for the full dispatcher)
- Build-time merge: `src/backend/access/nbtree/nbtsort.c:1153` in `_bt_load`
  - `deduplicate = wstate->inskey->allequalimage && !btspool->isunique && BTGetDeduplicateItems(wstate->index);`
  - the sorted input is folded into posting lists by `_bt_sort_dedup_finish_pending` (`nbtsort.c:1031`)
  - the `!btspool->isunique` term is why a unique index is never deduplicated on this path
- Storage parameter: `src/backend/access/common/reloptions.c:161` — `deduplicate_items`, default `true`, taken with `ShareUpdateExclusiveLock` because it applies only to later inserts
- VACUUM's index pass: `src/backend/access/nbtree/nbtree.c` — `btbulkdelete` removes dead entries and does not merge live ones

---

← Previous: [14_index_only_scan.md](14_index_only_scan.md) | Back to [README](README.md)
