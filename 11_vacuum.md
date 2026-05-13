# 11. VACUUM cycle — lazy_scan_heap → btbulkdelete → VM update

Thesis: §4.4.1
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
- **`tuples: 2 removed`** — only 2 heap tuples actually freed (the dead versions from earlier UPDATEs that hadn't been pruned by HOT). The 10 DELETEd rows had already been line-pointer-pruned during a prior scan via `kill_prior_tuple` opportunism.
- **`index scan needed: 3 pages from table had 12 dead item identifiers removed`** — twelve `LP_DEAD` line pointers across 3 heap pages triggered a single index-vacuuming pass. The per-AM callback `ambulkdelete` (here: `btbulkdelete` in `nbtree.c`) is invoked once per index, scans every leaf, and removes any entry whose TID is in the dead-TID list.
- **`index scans: 1`** — VACUUM keeps a list of dead TIDs in memory and does **one** pass per index, regardless of how many heap pages it scanned.
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

## Source path

- `src/backend/commands/vacuum.c:vacuum`
- → `src/backend/access/heap/vacuumlazy.c:heap_vacuum_rel`
  - → `lazy_scan_heap` (collects dead TIDs, sets HEAP_XMIN_COMMITTED hint bits)
  - → `lazy_vacuum_all_indexes`
    - → `src/backend/access/index/indexam.c:index_bulk_delete`
    - → `src/backend/access/nbtree/nbtree.c:btbulkdelete` (per-index callback for btree)
  - → `lazy_vacuum_heap_rel` (reclaim heap line pointers)
  - → `visibilitymap_set` (mark page all_visible, possibly all_frozen)

---

← Previous: [10_dirty_pages_comparison.md](10_dirty_pages_comparison.md) | Next: [12_bottom_up_deletion.md](12_bottom_up_deletion.md) →
