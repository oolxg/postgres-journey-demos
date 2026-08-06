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

## Inside a btree leaf

```sql
SELECT itemoffset, ctid, itemlen, substring(data for 20) AS data_first20
FROM bt_page_items('person_zipcode_idx', 1)
LIMIT 5;
```

```
 itemoffset |   ctid    | itemlen |        data_first20
------------+-----------+---------+-------------------------
          1 | (16,1)    |      16 | 0d 31 30 30 31 31 00
          2 | (16,8290) |     608 | 0d 31 30 30 30 30 00
          3 | (72,108)  |      16 | 0d 31 30 30 30 30 00
          4 | (73,72)   |      16 | 0d 31 30 30 30 30 00
          5 | (16,8291) |     616 | 0d 31 30 30 30 31 00
```

Items 2 and 5 are **posting tuples**: 608 / 616 bytes ≈ 1 key + ~100 TIDs (each TID = 6 bytes). Their `ctid` is the encoded posting-list header, not a real heap location (note the high offsets 8290/8291). Items 1, 3, 4 are regular 16-byte entries with one TID each — stragglers from inserts that arrived after the last dedup pass, or singletons that didn't trigger a pass.

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
