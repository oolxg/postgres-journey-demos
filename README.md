# postgres-journey-demos

Reproducible demo kit for **Processing Simple Queries: A Source-Level Study of PostgreSQL’s Implementation Details** (TU Berlin Bachelor thesis, DIMA, 2026). Companion to thesis Chapter 4 — every file here corresponds to one or more subsections in the chapter.

The thesis takes the prose; this repo takes the queries and the captured outputs.

## Prerequisites

- PostgreSQL 18.1 server.
- A database with the five contrib extensions installed:

```sql
CREATE DATABASE journey;
\c journey
CREATE EXTENSION pageinspect;
CREATE EXTENSION pg_buffercache;
CREATE EXTENSION pg_visibility;
CREATE EXTENSION pg_walinspect;
CREATE EXTENSION pgstattuple;
```

Then run [`00_setup.md`](00_setup.md) to create the demo table and populate it.

## Demos

Each demo file is self-contained: prose context, the SQL to run, the captured output, and a source-path block. Demos are numbered in pedagogical order, but most can run in isolation if the prerequisite state (mentioned in each demo's header) is set up first.

| # | File | Thesis subsection | Topic |
|---|------|-------------------|-------|
| 00 | [00_setup.md](00_setup.md) | §4.2.1 | Schema, populate, page layouts |
| 01 | [01_select_zipcode.md](01_select_zipcode.md) | §4.3.1, §4.3.2, §4.3.3 | SELECT by zipcode: parser → planner → BitmapHeapScan |
| 02 | [02_select_id_point.md](02_select_id_point.md) | §4.3.3 | SELECT by id: plain Index Scan |
| 03 | [03_seq_scan_baseline.md](03_seq_scan_baseline.md) | §4.3.2, §4.3.3 | Seq Scan baseline (no index on `age`) |
| 04 | [04_buffer_pool_walkthrough.md](04_buffer_pool_walkthrough.md) | §4.3.4 | Buffer pool contents via `pg_buffercache` |
| 04b | [04b_cold_cache.md](04b_cold_cache.md) | §4.3.5 | Server restart, cold→warm cache, 7× speedup |
| 05 | [05_insert_wal_trail.md](05_insert_wal_trail.md) | §4.4.1 | INSERT WAL trail (6 records per row) |
| 06 | [06_page_extension.md](06_page_extension.md) | §4.4.2 | Page extension when heap is full |
| 07 | [07_hot_update.md](07_hot_update.md) | §4.4.3 | HOT UPDATE (skips indexes) |
| 08 | [08_non_hot_update.md](08_non_hot_update.md) | §4.4.4 | non-HOT UPDATE (per-index entry) |
| 09 | [09_delete.md](09_delete.md) | §4.4.5 | DELETE (MVCC delegation) |
| 09b | [09b_kill_prior_tuple.md](09b_kill_prior_tuple.md) | §4.4.5 | `kill_prior_tuple` — scan-time LP_DEAD hint |
| 10 | [10_dirty_pages_comparison.md](10_dirty_pages_comparison.md) | §4.4.6 | Dirty-page signatures: HOT vs non-HOT vs DELETE |
| 11 | [11_vacuum.md](11_vacuum.md) | §4.5.1 | VACUUM cycle: `lazy_scan_heap` → `btbulkdelete` |
| 12 | [12_bottom_up_deletion.md](12_bottom_up_deletion.md) | §4.5.2 | Bottom-up deletion (PG14+) |
| 13 | [13_page_split.md](13_page_split.md) | §4.3.7, §4.4.7 | Page split, Lehman-Yao right-links |
| 14 | [14_index_only_scan.md](14_index_only_scan.md) | §4.3.6 | Index-Only Scan via VM, `Heap Fetches: 0` |
| 15 | [15_deduplication_control.md](15_deduplication_control.md) | §4.4.8 | Controlling dedup: incremental vs REINDEX vs off |

## Run order

For a clean run of any demo, reset with the queries in `00_setup.md`, then jump to the demo you want.

```sql
DROP TABLE IF EXISTS person;
-- copy the CREATE TABLE + CREATE INDEX from 00_setup.md
-- copy the INSERT INTO person + ANALYZE from 00_setup.md
-- run the demo's queries
```

Demos with a prerequisite state (e.g. Demo 07 needs `id=10100` inserted first) say so in their header.

## Notes on reproducibility

Captured output in each demo comes from a real run against the PG 18.1 build described below, on a `journey` database freshly populated via `00_setup.md`. When you run the same demos yourself, two classes of values behave differently:

**Reproduces exactly** — these are the claims the thesis rests on:

- WAL record *types* (e.g. `XLOG / FPI_FOR_HINT`, `Btree / INSERT_LEAF`, `Btree / DELETE`) — deterministic from the code path the SQL takes.
- Counts on deterministic workloads (e.g. `DELETE 10` → 10 dead heap tuples, 10 `LP_DEAD` bits; non-HOT UPDATE on indexed column → exactly 2 `Btree / INSERT_LEAF` records, one per index).
- EXPLAIN plan choice (Bitmap vs Index vs Seq Scan) on this schema and predicate — cost-based, statistics-stable.
- `usagecount` saturation at 5 (`BM_MAX_USAGE_COUNT`), tree level transitions (1→2 around 100k–200k inserts on pkey), heap page count for the 10000-row populator (74 pages), `Heap Fetches: 0` on the IOS path after VACUUM.
- Block numbers after splits (e.g. `412`, `411`, `698` in Demo 13) — deterministic for the 200k INSERT workload, provided the index starts from a freshly-populated 10000-row table and nothing else has extended it.

**Varies by run** — these are illustrative artefacts of one execution:

- `relfilenode` OIDs (e.g. `16812`, `16821`, `16823`) — assigned at `CREATE TABLE`/`CREATE INDEX` time, depend on the order of catalog inserts.
- Transaction IDs (`t_xmin = 769` etc.) — depend on the cluster's transaction history.
- WAL record sizes in bytes (e.g. `4458`, `7293`) — depend on whether an FPI is attached in the current checkpoint cycle.
- `bufferid` slot indices (e.g. `8223`, `8224`) — buffer manager runtime assignments.

The thesis's claims are about mechanism and category, not about specific OIDs. When numbers shift, the structure should still match: same WAL record types in the same order, the same leaf splits, the same page reclaim paths.

## Notes on the build

The thesis uses a PG 18.1 build with a small instrumentation patch on `catcache.c`, `tcop/postgres.c`, `parser/parse_relation.c`, `optimizer/util/plancat.c`, `optimizer/path/costsize.c`, `utils/adt/selfuncs.c`, and `executor/execMain.c` — a small diff adding `elog(NOTICE, ...)` lines to trace catalog access, planning, and execution. The patch is not required for the demos; on stock PG 18.1 the same queries produce the same outputs minus the `[CATCACHE …]`, `[STAGE …]`, `[PLANCAT …]`, `[COST …]`, and `[SELFUNCS …]` lines.

The full diff ships in this repository as [`elog_trace.patch`](elog_trace.patch), generated against the [`REL_18_1`](https://github.com/postgres/postgres/tree/REL_18_1) tag. What each touched file contributes:

| File (under `src/backend/` in the PostgreSQL tree) | Added trace |
|----------------------------------------------------|-------------|
| `tcop/postgres.c` | `[STAGE 1–3]` — parse, analyze+rewrite, plan, per statement in `exec_simple_query` |
| `executor/execMain.c` | `[STAGE 4 - EXECUTE]` — tuple count returned by `standard_ExecutorRun` |
| `parser/parse_relation.c` | `[parserOpenTable]` — table-name resolution |
| `utils/cache/catcache.c` | `[CATCACHE HIT/MISS/NEG-HIT]` — per-lookup outcome in `SearchCatCacheInternal` |
| `optimizer/util/plancat.c` | `[PLANCAT]` — pages/tuples/indexes read by `get_relation_info` |
| `utils/adt/selfuncs.c` | `[SELFUNCS]` — `pg_statistic` lookups in `examine_simple_variable` |
| `optimizer/path/costsize.c` | `[COST]` — cost arithmetic of `cost_seqscan` and `cost_index`, incl. selectivity and correlation |

The patch also adds `DIPLOMA`-prefixed explanatory comments at each stage; they change no behaviour. To reproduce the instrumented build:

```bash
git clone --branch REL_18_1 --depth 1 https://github.com/postgres/postgres.git
cd postgres
git apply /path/to/elog_trace.patch
./configure --prefix=$HOME/pg18-instrumented   # plus your usual options
make -j && make install
```

Only the instrumented-trace blocks in Demo 01 require this build.

The instrumented `selfuncs.c` path perturbs the planner's handling of `WHERE` clauses on function-scan RTE columns. Filters directly on `pg_buffercache_pages()` columns raise `bogus varno` on this build. The demos work around it by joining the `pg_buffercache` view against `pg_class` for filter conditions.

## License

MIT.
