# 11. VACUUM cycle — lazy_scan_heap → btbulkdelete → VM update

Thesis: §4.5.1
Prerequisite: [00_setup.md](00_setup.md), then run Demos 07–09 in the same session (so the heap has dead tuples and indexes have stale entries).

After UPDATEs and DELETEs, the heap and indexes carry dead/stale entries. VACUUM scans heap pages that aren't all_visible, collects dead TIDs, runs per-index `ambulkdelete` (`btbulkdelete` for btree), reclaims heap line pointers, and updates the visibility map.

## Pre-VACUUM state

`pgstattuple` reports per-relation tuple counts; `pg_visibility_map` reports the VM-bit state per page.

```sql
-- pgstattuple visits every tuple — accurate but expensive on big tables.
-- (s).field syntax dereferences the composite record returned by pgstattuple.
SELECT 'person'::text             AS rel,
       (s).tuple_count            AS live_tuples,
       (s).dead_tuple_count       AS dead_tuples,
       round((s).dead_tuple_percent::numeric, 2) AS dead_pct
FROM (SELECT pgstattuple('person') AS s) x
UNION ALL
SELECT 'person_zipcode_idx',
       (s).tuple_count, (s).dead_tuple_count,
       round((s).dead_tuple_percent::numeric, 2)
FROM (SELECT pgstattuple('person_zipcode_idx') AS s) x;

-- pg_visibility_map returns (all_visible, all_frozen) per heap page.
-- Group by both bits → distribution of pages by VM state.
SELECT all_visible, all_frozen, count(*) AS pages
FROM pg_visibility_map('person')
GROUP BY all_visible, all_frozen
ORDER BY all_visible, all_frozen;
```

```
        rel         | live_tuples | dead_tuples | dead_pct
--------------------+-------------+-------------+----------
 person             |        9992 |           2 |     0.02
 person_zipcode_idx |         334 |           0 |        0

 all_visible | all_frozen | pages
-------------+------------+-------
 f           | f          |     3       <-- pages with recent UPDATE/DELETE
 t           | f          |    71
```

3 heap pages are not all-visible — the pages affected by the recent UPDATEs and DELETEs.

## Run VACUUM

```sql
-- VERBOSE prints per-phase progress: heap scan, index bulk-delete, VM update.
VACUUM VERBOSE person;
```

```
INFO:  vacuuming "journey.public.person"
INFO:  finished vacuuming "journey.public.person": index scans: 1
pages: 0 removed, 74 remain, 3 scanned (4.05% of total)
tuples: 2 removed, 9931 remain, 0 are dead but not yet removable
removable cutoff: 778
frozen: 1 pages from table (1.35% of total) had 75 tuples frozen
visibility map: 3 pages set all-visible, 1 pages set all-frozen
index scan needed: 3 pages from table had 12 dead item identifiers removed
index "person_pkey":         pages: 30 in total, 0 newly deleted
index "person_zipcode_idx":  pages: 11 in total, 0 newly deleted
buffer usage: 81 hits, 34 reads, 13 dirtied
WAL usage: 18 records, 13 full page images, 91946 bytes
```

From this output:

- **`pages: 0 removed, 74 remain`** — VACUUM never shrinks the heap. PG's regular VACUUM only releases pages from the *end* of the relation (and only when fully empty); fragmented mid-relation pages are not given back. To physically shrink, run `VACUUM FULL` or `pg_repack`.
- **`tuples: 2 removed`** — only 2 heap tuples actually freed (the dead versions from earlier UPDATEs that hadn't been pruned by HOT). The 10 DELETEd rows had already been cut down to `LP_DEAD` stubs by on-access pruning (`heap_page_prune_opt`) when a prior scan read page 0. (`kill_prior_tuple` marks the matching *index* entries, not the heap line pointers.)
- **`index scan needed: 3 pages from table had 12 dead item identifiers removed`** — twelve `LP_DEAD` line pointers across 3 heap pages triggered a single index-vacuuming pass. The per-AM callback `ambulkdelete` (here: `btbulkdelete` in `nbtree.c`) is invoked once per index, scans every leaf, and removes any entry whose TID is in the dead-TID list.
- **`index scans: 1`** — VACUUM keeps a list of dead TIDs in memory and does **one** pass per index here, because all 12 dead TIDs fit in `maintenance_work_mem`. A dead-TID list that outgrows that budget forces additional passes, which is why the counter is reported at all.
- **`visibility map: 3 pages set all-visible, 1 pages set all-frozen`** — VM bits get updated, enabling future Index-Only Scans (Demo 14) to skip the heap.
- **WAL usage**: 18 records, 13 FPIs, ~92 KB. VACUUM is WAL-heavy because it touches many pages.

## Post-VACUUM state

Rerun the same `pgstattuple` + `pg_visibility_map` queries:

```
        rel         | live_tuples | dead_tuples | dead_pct
--------------------+-------------+-------------+----------
 person             |        9992 |           0 |        0
 person_zipcode_idx |         333 |           0 |        0    <- one entry gone

 all_visible | all_frozen | pages
-------------+------------+-------
 t           | f          |    73
 t           | t          |     1
```

`tuple_count` on `person_zipcode_idx` dropped from 334 to 333. `btbulkdelete` removed the index entries whose TIDs were in the dead-TID list: one of them had only that single dead TID (either a plain entry or a posting tuple shrunk to empty) and disappeared entirely.

The visibility map now reports **all 74 pages** as all-visible. Subsequent Index-Only Scans on `person` can answer queries from the index alone, skipping heap fetches entirely (Demo 14).

## Where the freed space goes

VACUUM does not shrink the table. After it frees a page's line pointers it records the
page's new free space in the free-space map (`vacuumlazy.c:2801`), and that is the map an
INSERT asks first (Demo 06). Rerunning Demo 06's 1000-row insert on a half-deleted table
shows the difference VACUUM makes. Both arms start from a fresh `00_setup` with
`autovacuum_enabled = off`, so nothing vacuums behind our back.

```sql
DELETE FROM person WHERE id % 2 = 0;        -- 5000 dead rows on every page
VACUUM person;                              -- ONLY in the second arm

-- run the insert from a NEW session: a backend caches its last target block
-- (hio.c:576) and only asks the FSM when it has none (hio.c:584)
INSERT INTO person SELECT g, 'Filler_' || g, (10000 + g % 100)::text, (20 + g % 60)::smallint
FROM generate_series(11001, 12000) g;

SELECT pg_relation_size('person') / 8192 AS heap_pages;
SELECT min((ctid::text::point)[0])::int AS min_blk,
       max((ctid::text::point)[0])::int AS max_blk,
       count(*) FILTER (WHERE (ctid::text::point)[0] < 74) AS on_old_pages
FROM person WHERE id > 11000;
```

Arm 1 — DELETE, **no** VACUUM, then the insert:

```
 heap_pages 
------------
         81
(1 row)

 min_blk | max_blk | on_old_pages 
---------+---------+--------------
      73 |      80 |           64
(1 row)
```

Arm 2 — DELETE, **VACUUM**, then the insert:

```
 heap_pages 
------------
         74
(1 row)

 min_blk | max_blk | on_old_pages 
---------+---------+--------------
       0 |      14 |         1000
(1 row)
```

Without VACUUM the dead rows still hold their slots, the FSM knows nothing, and the heap
grows by the same seven pages as in Demo 06 (the 64 rows that fit went to page 73, which
had room from the start). With VACUUM every new row lands in blocks 0-14, in the space the
DELETE left at the front of the file. Identical across three runs.

## VACUUM vs VACUUM FULL

Same half-deleted table, sizes in pages and the file each relation lives in:

```sql
SELECT relname, relfilenode, pg_relation_size(oid) / 8192 AS pages
FROM pg_class WHERE relname IN ('person','person_pkey','person_zipcode_idx')
ORDER BY relname;
```

After `00_setup`:

```
      relname       | relfilenode | pages 
--------------------+-------------+-------
 person             |       25954 |    74
 person_pkey        |       25963 |    30
 person_zipcode_idx |       25965 |    11
(3 rows)
```

After `DELETE FROM person WHERE id % 2 = 0; VACUUM person;` — same files, same sizes:

```
      relname       | relfilenode | pages 
--------------------+-------------+-------
 person             |       25954 |    74
 person_pkey        |       25963 |    30
 person_zipcode_idx |       25965 |    11
(3 rows)
```

After `VACUUM FULL person;` — three new files, all smaller:

```
      relname       | relfilenode | pages 
--------------------+-------------+-------
 person             |       25966 |    37
 person_pkey        |       25971 |    16
 person_zipcode_idx |       25972 |     7
(3 rows)
```

(relfilenode values differ per run; what matters is that plain VACUUM keeps them and
FULL replaces all three.) FULL is a variant of CLUSTER (`vacuum.c:2302`): it copies the
live rows into a new file and rebuilds every index.

The cost is the lock. `vacuum.c:2084` picks `ShareUpdateExclusiveLock` for plain VACUUM
and `AccessExclusiveLock` for FULL. With a reader holding a transaction open:

```sql
-- session A: an ordinary reader keeping its transaction open
BEGIN; SELECT count(*) FROM person; SELECT pg_sleep(6); COMMIT;

-- session B, while A sleeps
SET lock_timeout = '1s';
\timing on
VACUUM person;
VACUUM FULL person;
```

Session B output:

```
SET
Timing is on.
VACUUM
Time: 1.976 ms
ERROR:  canceling statement due to lock timeout
Time: 1002.368 ms (00:01.002)
```

(The `psql:<file>:4:` prefix that `psql -f` puts before the error line is omitted, since it
only names the local script file. `pg_locks` would show the two lock modes directly, but on the instrumented build the view
trips the known `could not open relation with OID 0` quirk, so it is not captured here.)

Plain VACUUM runs alongside readers and writers. FULL waits for everyone to leave.

## Source path

- `src/backend/commands/vacuum.c:vacuum`
- → `src/backend/access/heap/vacuumlazy.c:heap_vacuum_rel`
  - → `lazy_scan_heap` (collects dead TIDs, sets HEAP_XMIN_COMMITTED hint bits)
  - → `lazy_vacuum_all_indexes`
    - → `src/backend/access/index/indexam.c:index_bulk_delete`
    - → `src/backend/access/nbtree/nbtree.c:btbulkdelete` (per-index callback for btree)
  - → `lazy_vacuum_heap_rel` (reclaim heap line pointers, then `RecordPageWithFreeSpace` at line 2801)
  - → `visibilitymap_set` (mark page all_visible, possibly all_frozen)
- VACUUM FULL: `vacuum.c:2084` (lock choice) → `vacuum.c:2302` `cluster_rel` (rewrite into a new relfilenode)
- INSERT side: `src/backend/access/heap/hio.c:576` (cached target block) → `hio.c:584` `GetPageWithFreeSpace`

---

← Previous: [10_dirty_pages_comparison.md](10_dirty_pages_comparison.md) | Next: [12_bottom_up_deletion.md](12_bottom_up_deletion.md) →
