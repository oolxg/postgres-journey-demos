# 01. SELECT by zipcode — parser → planner → BitmapHeapScan

Thesis: §4.3.1, §4.3.2, §4.3.3
Prerequisite: [00_setup.md](00_setup.md) (74 heap pages, 10000 rows).

`zipcode = '10063'` matches 100 rows scattered across all 74 heap pages (`zipcode = 10000 + g % 100` → every 100th row shares a zipcode → uniform spread). The planner picks **Bitmap Heap Scan**: collect TIDs into a bitmap via the index, sort by heap block, then fetch heap pages in order — random I/O turned sequential.

## Plan, buffers, and time

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT name FROM person WHERE zipcode = '10063';
```

```
 Bitmap Heap Scan on public.person  (cost=5.06..84.23 rows=100 width=11) (actual time=0.035..0.145 rows=100.00 loops=1)
   Output: name
   Recheck Cond: (person.zipcode = '10063'::text)
   Heap Blocks: exact=74
   Buffers: shared hit=76
   ->  Bitmap Index Scan on person_zipcode_idx  (cost=0.00..5.04 rows=100 width=0) (actual time=0.016..0.016 rows=100.00 loops=1)
         Index Cond: (person.zipcode = '10063'::text)
         Index Searches: 1
         Buffers: shared hit=2
 Planning:
   Buffers: shared hit=93
 Planning Time: 0.418 ms
 Execution Time: 0.541 ms
```

Captured in a **fresh session against a warm buffer pool**: a new backend starts with an empty catcache, while the shared pool already holds the pages. That separates the catalog warm-up from disk I/O. Right after a server restart the same query reports `shared read=` instead of `hit=` and a different planning total, because the pool is cold and the first backend also rebuilds the relcache init file.

Three buffer counts:
- **Index buffers** `hit=2` — root (page 3) + 1 leaf.
- **Heap buffers** `hit=76` — 74 heap pages plus the child Bitmap Index Scan's 2 index buffers (EXPLAIN buffer counts are cumulative over child nodes). Every heap page had at least one matching row.
- **Planning buffers** `hit=93` — the first query of the session pays the catcache warm-up cost. The next section repeats the *same* query and shows the figure fall to nothing.

## Catcache warm-up: the same query twice

The comparison only isolates the catcache if nothing else changes, so this runs the **identical** query a second time in the **same session**:

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) SELECT name FROM person WHERE zipcode = '10063';  -- again
```

```
 Bitmap Heap Scan on public.person  (cost=5.06..84.23 rows=100 width=11) (actual time=0.019..0.072 rows=100.00 loops=1)
   Output: name
   Recheck Cond: (person.zipcode = '10063'::text)
   Heap Blocks: exact=74
   Buffers: shared hit=76
   ->  Bitmap Index Scan on person_zipcode_idx  (cost=0.00..5.04 rows=100 width=0) (actual time=0.009..0.009 rows=100.00 loops=1)
         Index Cond: (person.zipcode = '10063'::text)
         Index Searches: 1
         Buffers: shared hit=2
 Planning Time: 0.035 ms
 Execution Time: 0.234 ms
```

**There is no `Planning:` section at all.** EXPLAIN prints it only when planning touched at least one buffer, so its absence means the second plan was built entirely from the catcache: 93 buffer accesses down to zero. Planning time falls with it, 0.418 ms to 0.035 ms. Execution is unchanged (`hit=76`, `hit=2`) because the catcache has nothing to do with executing the plan.

A third identical run behaves like the second. The 93 reproduces across fresh sessions as long as the pool is warm.

## Catalog access pattern

With the diploma-instrumentation build (`NOTICE` `elog` lines across seven files — see [Notes on the build](README.md#notes-on-the-build)), the same query traces every catalog hit/miss:

```sql
SET client_min_messages = 'NOTICE';
SELECT name FROM person WHERE zipcode = '10063';
SET client_min_messages = 'WARNING';
```

The raw trace for this one statement is **103 lines: 29 misses and 65 hits over 12 distinct catalogs**. The excerpt below is filtered, and the filter is: every `STAGE`, `PLANCAT`, `SELFUNCS` and `COST` line, plus the *first* miss of each catalog. Repeated misses and all hits are dropped. Every line shown is verbatim.

```
[STAGE 1 - PARSE] Query: "SELECT name FROM person WHERE zipcode = '10063';" -> parsed into 1 raw statement(s)
[parserOpenTable] relName: person
[CATCACHE MISS] pg_namespace -> will read from disk
[CATCACHE MISS] pg_class -> will read from disk
[CATCACHE MISS] pg_index -> will read from disk
[CATCACHE MISS] pg_attribute -> will read from disk
[CATCACHE MISS] pg_am -> will read from disk
[CATCACHE MISS] pg_operator -> will read from disk
[CATCACHE MISS] pg_type -> will read from disk
[CATCACHE MISS] pg_proc -> will read from disk
[STAGE 2 - ANALYZE+REWRITE] Produced 1 query tree(s)
[PLANCAT] get_relation_info: person (oid=25347) pages=74 tuples=10000 indexes=2
[CATCACHE MISS] pg_statistic -> will read from disk
[SELFUNCS] examine_simple_variable: looked up pg_statistic for rel=25347 att=3 -> FOUND
[CATCACHE MISS] pg_tablespace -> will read from disk
[COST] SeqScan: rel=1 pages=74 tuples=10000 | startup=0.00 total=199.00 (disk=74.00 cpu=125.00)
[CATCACHE MISS] pg_amop -> will read from disk
[COST] IndexScan: rel=1 idx_pages=11 tbl_pages=74 tuples=10000 | selectivity=0.010000 corr=0.0194 fetched=100 | startup=0.29 total=245.95
[STAGE 3 - PLAN] Plan node tag: 344, est. rows: 100, total cost: 84.23
[STAGE 4 - EXECUTE] Returned 100 tuple(s)
```

Eleven catalogs miss the cache on this statement. Nine carry the plan decision: name resolution (`pg_namespace`, `pg_class`), index discovery (`pg_index`, `pg_am`), column shape (`pg_attribute`), operator and opclass resolution (`pg_operator`, `pg_proc`, `pg_amop`), type metadata (`pg_type`), and selectivity (`pg_statistic`). Two belong to statement setup rather than planning: `pg_tablespace` for storage parameters, and `pg_authid` for the permission check (the latter misses before the SELECT is even parsed, so it falls outside the excerpt).

Each `MISS` is a catcache miss followed by a heap read of the catalog. `HIT` costs nothing. Note how uneven the misses are: `pg_attribute` misses 8 times and `pg_class` and `pg_index` 5 each, because the catcache is keyed per lookup, not per catalog — a different column or a different index is a different key.

`pg_cast` does **not** appear on this statement: the predicate compares `text` to a `text` literal, so no cast has to be resolved.

The cost trace explains the plan choice. All three totals are for the full scan of the same predicate, so they compare directly:

- **SeqScan = 199.00.** Every page is read once and every row is tested:
  `74 × seq_page_cost(1.0) + 10000 × cpu_tuple_cost(0.01) + 10000 × cpu_operator_cost(0.0025) = 74 + 100 + 25`.
  The trace splits it the same way: `disk=74.00 cpu=125.00`.
- **IndexScan = 245.95** — *higher* than a full sequential scan. With correlation 0.0194 the 100 matching TIDs arrive in an order unrelated to physical layout, so the planner charges close to `random_page_cost(4.0)` per fetch instead of 1.0.
- **Bitmap Heap Scan = 84.23**, of which 5.06 is startup (build the bitmap through the index) and the rest is fetching 74 heap pages in block order. Sorting the TIDs first is what converts those random fetches back into sequential ones.

Beware when reproducing this: adding `LIMIT` changes the number. The planner then costs a plan that only needs a few rows, and the total drops far below 84.23 while `SeqScan` and `IndexScan` stay at their full-scan figures, because those are path costs computed below the limit.

## Source path

- Parser: `src/backend/parser/gram.y` → `pg_parse_query()` → `RawStmt` list
- Analyzer: `src/backend/parser/analyze.c:parse_analyze_fixedparams` — name resolution via syscache
- Rewriter: `src/backend/rewrite/rewriteHandler.c:QueryRewrite` (no-op for this query)
- Planner: `src/backend/optimizer/plan/planner.c:standard_planner` → `optimizer/path/allpaths.c:set_plain_rel_pathlist` builds SeqScan + IndexScan + BitmapHeapScan paths, picks cheapest
- Executor: `src/backend/executor/nodeBitmapHeapscan.c:ExecBitmapHeapScan` → `nodeBitmapIndexscan.c` → `access/nbtree/nbtree.c:btgetbitmap` (descends from `fastroot`, collects TIDs into a `TIDBitmap`) → heap fetches in block order

---

← Previous: [00_setup.md](00_setup.md) | Next: [02_select_id_point.md](02_select_id_point.md) →
