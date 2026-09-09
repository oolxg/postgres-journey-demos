# 00. Setup and Populate

Thesis: §4.2.1

Schema is two-index, mixed-cardinality. `id` is monotonically ascending (pkey, no duplicates). `zipcode` is non-unique with ~100 distinct values, each repeating ~100 times — heavy duplication, on purpose, to exercise btree deduplication later.

## Schema

```sql
DROP TABLE IF EXISTS person;

CREATE TABLE person (
    id      bigint   PRIMARY KEY,    -- monotonic ascending btree
    name    text     NOT NULL,       -- NOT indexed → HOT update target
    zipcode text     NOT NULL,       -- non-unique btree, ~100 distinct values
    age     smallint NOT NULL        -- NOT indexed → Seq Scan baseline (Demo 03)
);

CREATE INDEX person_zipcode_idx ON person (zipcode);
```

Initial state — empty heap (0 bytes), each btree is just one metapage:

```sql
SELECT relname,
       pg_relation_size(oid)        AS bytes,
       pg_relation_size(oid) / 8192 AS pages
FROM pg_class
WHERE relname IN ('person', 'person_pkey', 'person_zipcode_idx')
ORDER BY relname;
```

```
      relname       | bytes | pages
--------------------+-------+-------
 person             |     0 |     0
 person_pkey        |  8192 |     1
 person_zipcode_idx |  8192 |     1
```

Both indexes already occupy 1 page — the **metapage** (page 0): magic number, format version, root pointer (currently zero), tree level, and `fastroot`. The root leaf is allocated lazily, on the first INSERT.

## Populate

10000 rows. `id` runs 1..10000, monotonic. `zipcode` is drawn from a pool of 100 values (`'10000'..'10099'`), each repeating ~100 times.

```sql
INSERT INTO person (id, name, zipcode, age)
SELECT
    g                          AS id,
    'Person_' || g             AS name,
    (10000 + (g % 100))::text  AS zipcode,
    (20 + (g % 60))::smallint  AS age
FROM generate_series(1, 10000) g;

ANALYZE person;
```

State after populate + ANALYZE:

```sql
SELECT relname,
       pg_relation_size(oid)        AS bytes,
       pg_relation_size(oid) / 8192 AS pages,
       reltuples::int               AS planner_rows
FROM pg_class
WHERE relname IN ('person', 'person_pkey', 'person_zipcode_idx')
ORDER BY relname;
```

```
      relname       | bytes  | pages | planner_rows
--------------------+--------+-------+--------------
 person             | 606208 |    74 |        10000
 person_pkey        | 245760 |    30 |        10000
 person_zipcode_idx |  90112 |    11 |        10000
```

`person_pkey` is 30 pages for 10000 unique bigints. `person_zipcode_idx` is only **11 pages** — almost three times smaller despite holding the same number of TIDs. Reason: **deduplication** (`nbtdedup.c`, PG13+). When a leaf accumulates duplicate keys, btree merges them into one entry with a posting list — one key, many TIDs.

Tree height:

```sql
SELECT 'pkey'    AS idx, level, root, fastroot FROM bt_metap('person_pkey')
UNION ALL
SELECT 'zipcode', level, root, fastroot FROM bt_metap('person_zipcode_idx');
```

```
   idx   | level | root | fastroot
---------+-------+------+----------
 pkey    |     1 |    3 |        3
 zipcode |     1 |    3 |        3
```

Both at `level = 1` (root + leaves). No internal pages yet — the root fits all leaf-level separators in one page.

## Why 10000 rows

The size is not arbitrary. Below roughly a thousand rows the structures this kit inspects do not exist yet. Same schema, same populator formulas, three sizes:

```sql
-- repeated for n in (100, 1000, 10000) against a probe copy of the schema
SELECT pg_relation_size('probe')/8192        AS heap_pages,
       pg_relation_size('probe_pkey')/8192   AS pkey_pages,
       (SELECT level FROM bt_metap('probe_pkey'))    AS pkey_level,
       pg_relation_size('probe_zip_idx')/8192 AS zip_pages,
       (SELECT level FROM bt_metap('probe_zip_idx')) AS zip_level;
```

```
  rows | heap_pages | pkey_pages | pkey_level | zip_pages | zip_level | posting_tuples | max_itemlen
-------+------------+------------+------------+-----------+-----------+----------------+-------------
   100 |          1 |          2 |          0 |         2 |         0 |              0 |          16
  1000 |          8 |          5 |          1 |         4 |         1 |            100 |          80
 10000 |         74 |         30 |          1 |        11 |         1 |            100 |         616
```

Three things only appear at the larger size:

- **A tree at all.** At 100 rows both indexes report `level = 0`: the root *is* the only leaf. There are no internal pages, no descent, no high keys, no right-links, and a page split cannot happen. Demos 13 and 14 would have nothing to show.
- **A heap worth scanning.** At 100 rows the heap is a single page, so Seq Scan, Index Scan and Bitmap Heap Scan all cost one page fetch and the plan comparison of Demos 01-03 is meaningless. At 10000 rows the heap is 74 pages and the 100 rows matching one zipcode are spread across all of them, which is what makes the bitmap reorder pay off.
- **Duplicates to merge.** `zipcode = 10000 + (g % 100)` gives `n / 100` rows per key. At `n = 100` every zipcode occurs exactly once, so there is nothing to deduplicate and `posting_tuples` is 0. At `n = 1000` posting tuples appear but hold ~10 TIDs each (80 bytes) and save only one index page. At `n = 10000` each holds ~100 TIDs (616 bytes) and the index drops from 30 pages to 11.

10000 is the smallest round size where all three hold at once. Going much higher costs more than it buys: the three relations already occupy 920 KB, the whole workload rebuilds in about a second, and every demo runs in milliseconds, while a dump of one index leaf still fits on a screen. A hundred thousand rows would change none of the mechanisms.

Where a mechanism really does need more rows, the demo that needs them adds them: the primary key only reaches tree level 2 after roughly 200000 further inserts, which is what Demo 13 does.

## Inside a btree leaf

`bt_page_items` returns the raw bytes of each index entry in a `data` column. That column is a hex dump, because the function works on any index over any type and cannot know how to print an arbitrary key. For a `text` key the bytes are a one-byte length header followed by the characters, so the query below decodes them back into the zipcode:

```sql
SELECT itemoffset,
       itemlen,
       coalesce(array_length(tids, 1), 1)                     AS row_pointers,
       convert_from(decode(replace(substring(data from 4 for 14), ' ', ''),
                           'hex'), 'UTF8')                    AS zipcode
FROM bt_page_items('person_zipcode_idx', 1)
LIMIT 5;
```

```
 itemoffset | itemlen | row_pointers | zipcode 
------------+---------+--------------+---------
          1 |      16 |            1 | 10011
          2 |     608 |           98 | 10000
          3 |      16 |            1 | 10000
          4 |      16 |            1 | 10000
          5 |     616 |           99 | 10001
```

Read it by column. `itemlen` is the size of the entry. `row_pointers` is how many table rows the entry points at. `zipcode` is the decoded key.

Two shapes are visible. Items 2 and 5 are **posting tuples**: one key, 98 and 99 row pointers, 608 and 616 bytes. Items 1, 3 and 4 are plain entries: one key, one row pointer, 16 bytes.

Items 2, 3 and 4 all hold the key `10000`. That key is assigned whenever `g % 100 = 0`, so 100 rows carry it. The posting tuple holds 98 of them and the two plain entries hold the last two: 98 + 2 = 100. Those two arrived after the last dedup pass on this leaf and wait for the next one.

Item 1 is the page **high key**, the upper bound on keys that belong here. Page 1 is not the rightmost leaf, so it stores one.

Two notes on the query. The `tids` column is null for a plain entry, which is why `coalesce` maps it to 1. And adding `WHERE itemoffset IN (2,5)` raises `could not open relation with OID 0` on the instrumented build (see README), so the query uses `LIMIT 5` instead.

Dedup is **opportunistic, not eager** — `_bt_dedup_pass` runs only when a leaf is about to split and adjacent duplicates exist. So a leaf in steady state is a mix of posting tuples (from past dedup passes) and plain entries (stragglers since the last pass).

## Inside a heap page

```sql
SELECT lp, lp_off, lp_flags, lp_len, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('person', 0))
LIMIT 5;
```

```
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_ctid
----+--------+----------+--------+--------+--------+--------
  1 |   8136 |        1 |     50 |    769 |      0 | (0,1)
  2 |   8080 |        1 |     50 |    769 |      0 | (0,2)
  3 |   8024 |        1 |     50 |    769 |      0 | (0,3)
  4 |   7968 |        1 |     50 |    769 |      0 | (0,4)
  5 |   7912 |        1 |     50 |    769 |      0 | (0,5)
```

`lp_flags = 1` means `LP_NORMAL` — a live tuple. The four possible line-pointer states (`src/include/storage/itemid.h`):

- `LP_UNUSED` — slot unused, free to allocate
- `LP_NORMAL` — points at a live tuple
- `LP_REDIRECT` — HOT-prune redirect, points at the next live tuple in the chain
- `LP_DEAD` — the tuple is known dead (set by HOT pruning or by VACUUM — `kill_prior_tuple` sets LP_DEAD on *index* items, not on heap line pointers); the slot is reclaimable but the data is still around

`t_xmin = 769` — the xid that inserted the row. `t_xmax = 0` — not deleted. `t_ctid = (0,1)` — self pointer; this *is* the current version.

`t_data` packs `id`, `name`, `zipcode`, `age` in declaration order, with varlena headers on the two text columns and alignment padding before `age` (smallint).

## Source paths

- Heap storage format: `src/include/storage/bufpage.h`, `src/include/access/htup_details.h`
- btree page format: `src/backend/access/nbtree/README`, `src/include/access/nbtree.h` (`BTPageOpaqueData`, posting-list encoding)
- Dedup: `src/backend/access/nbtree/nbtdedup.c` (`_bt_dedup_pass`)
- Line pointer states: `src/include/storage/itemid.h`

---

Next: [01_select_zipcode.md](01_select_zipcode.md) →
