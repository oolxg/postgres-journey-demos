# 01. SELECT by zipcode — parser → planner → BitmapHeapScan

Thesis: §4.2.1, §4.2.2, §4.2.3
Prerequisite: [00_setup.md](00_setup.md) (74 heap pages, 10000 rows).

`zipcode = '10063'` matches 100 rows scattered across all 74 heap pages (`zipcode = 10000 + g % 100` → every 100th row shares a zipcode → uniform spread). The planner picks **Bitmap Heap Scan**: collect TIDs into a bitmap via the index, sort by heap block, then fetch heap pages in order — random I/O turned sequential.

## Plan, buffers, and time

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, COSTS OFF)
SELECT name FROM person WHERE zipcode = '10063';
```

```
 Bitmap Heap Scan on public.person (actual time=0.096..0.347 rows=100.00 loops=1)
   Output: name
   Recheck Cond: (person.zipcode = '10063'::text)
   Heap Blocks: exact=74
   Buffers: shared hit=76
   ->  Bitmap Index Scan on person_zipcode_idx (actual time=0.038..0.038 rows=100.00 loops=1)
         Index Cond: (person.zipcode = '10063'::text)
         Index Searches: 1
         Buffers: shared hit=2
 Planning:
   Buffers: shared hit=96 dirtied=3
 Planning Time: 1.544 ms
 Execution Time: 1.217 ms
```

Three buffer counts:
- **Index buffers** `hit=2` — root (page 3) + 1 leaf.
- **Heap buffers** `hit=76` — bitmap heap scan touched 74 heap pages + 2 metadata pages. Every heap page had at least one matching row.
- **Planning buffers** `hit=96 dirtied=3` — first query of the session pays the catcache warm-up cost. Subsequent queries amortise (Demo 03 will show planning down to ~12).

## Catalog access pattern

With the diploma-instrumentation build (`NOTICE` `elog` in `catcache.c` / `tcop/postgres.c`), the same query traces every catalog hit/miss:

```sql
SET client_min_messages = 'NOTICE';
SELECT name FROM person WHERE zipcode = '10063' LIMIT 5;
SET client_min_messages = 'WARNING';
```

Trace excerpt:

```
[STAGE 1 - PARSE] Query: "SELECT name FROM person WHERE zipcode = '10063' LIMIT 5;"
                  -> parsed into 1 raw statement(s)
[parserOpenTable] relName: person
[CATCACHE MISS] pg_namespace -> will read from disk      -- "public" schema lookup
[CATCACHE MISS] pg_class    -> will read from disk       -- "person" table lookup
[CATCACHE MISS] pg_index    -> will read from disk       -- indexes on person
[CATCACHE MISS] pg_attribute-> will read from disk       -- column metadata
[CATCACHE MISS] pg_operator -> will read from disk       -- "=" operator
[CATCACHE MISS] pg_type     -> will read from disk       -- text type
[CATCACHE MISS] pg_proc     -> will read from disk       -- comparison procs
[CATCACHE MISS] pg_cast     -> will read from disk       -- text casts
[STAGE 2 - ANALYZE+REWRITE] Produced 1 query tree(s)
[PLANCAT] get_relation_info: person (oid=16498) pages=74 tuples=10000 indexes=2
[CATCACHE MISS] pg_statistic -> will read from disk      -- per-column statistics
[SELFUNCS] examine_simple_variable: looked up pg_statistic for rel=16498 att=3 -> FOUND
[COST] SeqScan:   rel=1 pages=74 tuples=10000 | startup=0.00 total=199.00 (disk=74.00 cpu=125.00)
[COST] IndexScan: rel=1 idx_pages=11 tbl_pages=74 tuples=10000 | selectivity=0.010000 corr=0.0194 fetched=100 | startup=0.29 total=245.95
[STAGE 3 - PLAN] BitmapHeapScan chosen, est. rows: 5, total cost: 9.02
[STAGE 4 - EXECUTE] Returned 5 tuple(s)
```

Ten catalogs across six purposes: name resolution (`pg_namespace`, `pg_class`), index discovery (`pg_index`, `pg_am`), column shape (`pg_attribute`), operator resolution (`pg_operator`, `pg_proc`, `pg_cast`), type metadata (`pg_type`), and selectivity (`pg_statistic`). Each `MISS` is a syscache miss followed by a heap read of the catalog; `HIT` is a syscache hit (no I/O).

The cost trace explains the plan choice:
- SeqScan = 199.00 (74 pages × `seq_page_cost` + 10000 × `cpu_tuple_cost`)
- IndexScan = 245.95 — *higher* than SeqScan because random-ordered TID fetches across 74 heap pages dominate
- Bitmap Heap Scan reorders the fetches and lands at total ≈ 9.02

## Source path

- Parser: `src/backend/parser/gram.y` → `pg_parse_query()` → `RawStmt` list
- Analyzer: `src/backend/parser/analyze.c:parse_analyze_fixedparams` — name resolution via syscache
- Rewriter: `src/backend/rewrite/rewriteHandler.c:QueryRewrite` (no-op for this query)
- Planner: `src/backend/optimizer/plan/planner.c:standard_planner` → `optimizer/path/allpaths.c:set_plain_rel_pathlist` builds SeqScan + IndexScan + BitmapHeapScan paths, picks cheapest
- Executor: `src/backend/executor/nodeBitmapHeapscan.c:ExecBitmapHeapScan` → `nodeBitmapIndexscan.c` → `access/nbtree/nbtree.c:btgetbitmap` (descends from `fastroot`, collects TIDs into a `TIDBitmap`) → heap fetches in block order

---

← Previous: [00_setup.md](00_setup.md) | Next: [02_select_id_point.md](02_select_id_point.md) →
