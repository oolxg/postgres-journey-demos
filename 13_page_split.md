# 13. Page split — tree height grows + Lehman-Yao right-links

Thesis: §4.5.1, §4.5.2
Prerequisite: [00_setup.md](00_setup.md). Run on a fresh DB.

Both indexes start at level 1 (root + leaves). Inserting ~200k more rows on monotonically increasing `id` forces leaf splits in `person_pkey`. When the root's separator slots overflow, a new root page is allocated above and the tree gains a level. Internal pages also have `btpo_next` right-links so a concurrent reader pinned on an OLD page can still find its way (Lehman-Yao).

## Tree level before / after

```sql
-- bt_metap returns the metapage of a btree — root pointer, current level,
-- fastroot shortcut. pageinspect contrib.
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

Insert 200001 more rows:

```sql
-- Monotonic id (continues from 100000+) → every insert lands on the rightmost
-- leaf of pkey → that leaf fills, splits, parent eventually overflows too.
INSERT INTO person (id, name, zipcode, age)
SELECT g, 'P_' || g, '30' || lpad((g % 999)::text, 3, '0'), 30
FROM generate_series(100000, 300000) g;
```

Re-run the same `bt_metap` UNION:

```
   idx   | level | root | fastroot
---------+-------+------+----------
 pkey    |     2 |  412 |      412     <-- new root, level 2
 zipcode |     1 |    3 |        3     <-- still level 1 (dedup keeps it dense)
```

`person_pkey` grew to **level 2**: a root + an internal level + leaves. The new root lives at page 412. The old root (page 3) became an internal page.

## Inspect the new root and the demoted old root

```sql
-- bt_page_stats decodes a btree page into header fields: type (r/i/l), live items,
-- right/left links, page level, flags. type='r' = root, 'i' = internal, 'l' = leaf.
SELECT * FROM bt_page_stats('person_pkey', 412);

SELECT * FROM bt_page_stats('person_pkey', 3);
```

```
 blkno | type | live_items | btpo_level | btpo_flags
-------+------+------------+------------+------------
   412 |  r   |          3 |          2 |          2

 blkno | type | live_items | btpo_level | btpo_next
-------+------+------------+------------+----------
     3 |  i   |        286 |          1 |       411
```

Page 412 is the new root: 3 items pointing at 3 internal pages. Page 3, the former root, is now an internal page (`type=i`) with 286 separator entries. Its `btpo_next=411` is the right-link to its sibling internal page.

The full internal-level right-link chain:

```sql
SELECT * FROM bt_page_stats('person_pkey', 411);
SELECT * FROM bt_page_stats('person_pkey', 698);
```

```
 blkno | type | live_items | btpo_prev | btpo_next
-------+------+------------+-----------+----------
   411 |  i   |        286 |         3 |       698
   698 |  i   |        141 |       411 |         0
```

Reading the chain: page 3 → page 411 → page 698 → NIL (`btpo_next=0`). All three internal pages also carry `btpo_prev` so the chain can be walked in either direction.

```sql
-- bt_page_items decodes per-entry on a single btree page.
-- First 8 bytes of `data` for an internal page = pointer to child page; for a
-- leaf = the key bytes.
SELECT itemoffset, ctid, itemlen, substring(data for 8) AS key_first_8_bytes
FROM bt_page_items('person_pkey', 412);
```

```
 itemoffset |  ctid   | itemlen |  key_first_8_bytes
------------+---------+---------+-------------------------
          1 | (3,0)   |       8 |               (none)
          2 | (411,1) |      16 | b3 33 02 00 00 00 00 00     -- bigint 144819
          3 | (698,1) |      16 | 29 cb 03 00 00 00 00 00     -- bigint 248745
```

Internal-page items map: `(3,0)` — leftmost, no key (everything below the first separator goes here); `(411,1)` — second separator key 144819, children with keys ≥ 144819 go here; `(698,1)` — third separator key 248745.

```
                                     ┌─────────────────────────────────────────┐
                                     │  metapage (page 0)                      │
                                     │  root=412, level=2, fastroot=412        │
                                     └───────────────────┬─────────────────────┘
                                                         │
                                                         ▼
                                  ┌─────────────────────────────────────────┐
                                  │      NEW ROOT page 412 (level=2)        │
                                  │  ┌───────┬─────────────┬─────────────┐  │
                                  │  │ → (3) │ 144819→(411)│ 248745→(698)│  │
                                  │  └───────┴─────────────┴─────────────┘  │
                                  └─────┬──────────────┬───────────────┬────┘
                                        │              │               │
        ┌───────────────────────────────┘              │               └───────────────────────────────┐
        ▼                                              ▼                                               ▼
┌───────────────────────┐ ─btpo_next→ ┌───────────────────────┐ ─btpo_next→ ┌───────────────────────┐
│ INTERNAL page 3       │             │ INTERNAL page 411     │             │ INTERNAL page 698     │
│  level=1              │ ←btpo_prev─ │  level=1              │ ←btpo_prev─ │  level=1              │
│  286 separators       │             │  286 separators       │             │  141 separators       │
└──────────┬────────────┘             └──────────┬────────────┘             └──────────┬────────────┘
           │                                     │                                     │
           ▼                                     ▼                                     ▼
        ...leaves...                          ...leaves...                          ...leaves...
```

- The root **moved** to a new page (412). The old root (page 3) was demoted in place — its contents stayed; what changed is the metapage now points at 412 and page 3 has `btpo_level = 1` (internal, not root).
- Right-links exist at every level. Internal pages 3, 411, 698 are linked left-to-right via `btpo_next`. Leaves under each have their own right-link chain.
- Reading `bt_metap('person_pkey')` returns the **current** root pointer (412). The metapage's `fastroot` field can shortcut to a deeper level if there are no useful upper levels (see Demo 00); for a freshly split 2-level tree fastroot equals root.

## Why right-links matter — Lehman-Yao read with concurrent split

A reader R is descending the tree to find key 175000. It pinned leaf L (which holds keys 144800..200000). Before R reads it, a writer W splits L into L (keeps 144800..172000) and L' (gets 172001..200000), updating the parent atomically.

```
Before split:                    After split (parent already updated):
   parent: [..., k=144800 → L]      parent: [..., k=144800 → L, k=172001 → L', ...]

   L                                L                                L'
   ┌─────────────────────┐          ┌─────────────────────┐          ┌─────────────────────┐
   │ keys 144800..200000 │ → NIL    │ keys 144800..172000 │ ──────▶  │ keys 172001..200000 │ → NIL
   │ high_key = 200000   │          │ high_key = 172000   │          │ high_key = 200000   │
   └─────────────────────┘          └─────────────────────┘          └─────────────────────┘
```

Reader R is still pinned on L. It checks: target key 175000 vs L's high_key (now 172000). Since 175000 > 172000, R follows the right-link to L' and finds the key there. **No locking, no re-traversal from the root.** That is the Lehman-Yao invariant: every page knows its right neighbour, and the high_key tells the reader when to step right.

## Source path

- `src/backend/access/nbtree/nbtinsert.c:_bt_doinsert` — top of the insert call chain
- `src/backend/access/nbtree/nbtinsert.c:_bt_findinsertloc` — leaf-full check
- `src/backend/access/nbtree/nbtinsert.c:_bt_split` — divides entries across two leaves; sets `btpo_next` on the old leaf BEFORE releasing the write lock (Lehman-Yao invariant)
- `_bt_insert_parent` recurses upward; at the root, `_bt_newlevel` (`nbtinsert.c:2426`) allocates a brand-new root page and leaves the old root in place
- Right-link traversal during concurrent reads: `src/backend/access/nbtree/nbtsearch.c:_bt_moveright`

(Block numbers above are deterministic for the 200k INSERT workload on a freshly-populated 10k-row table; adjust expectations if you run a different load.)

---

← Previous: [12_bottom_up_deletion.md](12_bottom_up_deletion.md) | Next: [14_index_only_scan.md](14_index_only_scan.md) →
