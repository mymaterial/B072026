# random_page_cost, Bitmap Heap Scans, Parallel Workers & huge_pages — Live Demos

Completing the PgTune parameter walkthrough from the earlier session: a real before/after `random_page_cost` demo with actual timing numbers, what a bitmap heap scan physically does (and why `effective_io_concurrency` only matters for it), the parallel-worker parameters confirmed live with real "Workers Planned vs. Launched" numbers, a live `huge_pages` troubleshooting session with real error messages, and the short, blunt list of which parameters you'll actually touch again after go-live.

---
## Table of Contents

- [1. random_page_cost — the Live Before/After](#1-random_page_cost--the-live-beforeafter)
- [2. Why SSD Changes the Answer](#2-why-ssd-changes-the-answer)
- [3. Query Hints Are a Symptom, Not a Fix](#3-query-hints-are-a-symptom-not-a-fix)
- [4. What a Bitmap Heap Scan Actually Does](#4-what-a-bitmap-heap-scan-actually-does)
- [5. effective_io_concurrency — Only Matters for Bitmap Scans](#5-effective_io_concurrency--only-matters-for-bitmap-scans)
- [6. max_worker_processes, max_parallel_workers, max_parallel_workers_per_gather — Confirmed Live](#6-max_worker_processes-max_parallel_workers-max_parallel_workers_per_gather--confirmed-live)
- [7. Parallelism Only Applies to SELECT](#7-parallelism-only-applies-to-select)
- [8. max_parallel_maintenance_workers — Confirmed on CREATE INDEX](#8-max_parallel_maintenance_workers--confirmed-on-create-index)
- [9. huge_pages — Live Troubleshooting](#9-huge_pages--live-troubleshooting)
- [10. The Short List — What You'll Actually Touch Again](#10-the-short-list--what-youll-actually-touch-again)
- [11. pgconfig.org — an Alternative to PgTune](#11-pgconfigorg--an-alternative-to-pgtune)
- [12. Questions Discussed in This Session](#12-questions-discussed-in-this-session)

---

## 1. random_page_cost — the Live Before/After

Against a large indexed table (`demo`), the same query run twice, changing only `random_page_cost`:

```sql
explain analyze select * from demo where val between 100000 and 150000;
```
**At the default, higher `random_page_cost` (effectively telling the planner random I/O is expensive):**
```
Gather (actual time=99.878..12689.665 rows=250025.00 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  -> Parallel Bitmap Heap Scan on demo ...
Execution Time: 12716.821 ms
```

```sql
set random_page_cost = 1.1;
explain analyze select * from demo where val between 100000 and 150000;
```
**Same query, same data, only `random_page_cost` changed:**
```
Index Scan using ix on demo (actual time=0.105..1079.494 rows=250025.00 loops=1)
Execution Time: 1094.826 ms
```

**The result: ~12,717ms down to ~1,095ms — roughly a 11–12× speedup, for the identical query against the identical data, changing exactly one parameter.** This is about as clean a live demonstration as this course produces of a single config value materially changing real query performance — the plan itself switched from a parallel bitmap heap scan to a plain index scan, and execution time collapsed accordingly.

---

## 2. Why SSD Changes the Answer

The underlying logic, stated directly: on a **hard disk drive**, jumping to random, scattered locations on the physical platter is genuinely expensive — hence the historical default of `random_page_cost = 4` (random access costed as 4× a sequential read). On an **SSD**, there's no physical head to move — random and sequential access cost roughly the same, which is exactly why `random_page_cost = 1.1` (barely above `seq_page_cost`'s `1.0`) is the correct value on modern storage: **it's telling the planner "I'm on fast disk, feel free to prefer the index."**

**Confirmed directly: "99.999% of cases nowadays are all SSD"** — this is why PgTune ships `1.1` as its default recommendation rather than the historical `4`.

---

## 3. Query Hints Are a Symptom, Not a Fix

**A direct point made about optimizer hints** (raised via a student's own prior experience with `INDEX`/parallel hints): forcing a specific plan via a hint is functionally the same thing as what `random_page_cost` does, just scoped to a single query instead of the whole instance — *"you are forcing your optimizer to go for index scan instead of sequential scan."* **The broader lesson:** if you find yourself repeatedly hinting queries toward index scans, that's often a signal the underlying cost parameters (`random_page_cost`, `effective_io_concurrency`) are miscalibrated for your actual storage, and fixing them at the instance level is usually the better fix than hinting every affected query individually.

---

## 4. What a Bitmap Heap Scan Actually Does

Walked through with a physical, step-by-step comparison of all three scan types, using a small worked example (an `emp` table with an index on `id`):

**Plain index scan (few matching rows, scattered):** for each matching value, go to the index, find its physical location, jump straight to the table, fetch that row, put it in memory — repeat per row. Described directly: *"repeated index and table scan put together."*

**Bitmap heap scan (many matching rows, scattered across the table):**
1. Scan the **index first**, collecting *every* matching row's physical location — not fetching the actual row data yet, just the locations.
2. Store those collected locations as a **bitmap** — *"a collection of pointers in a file that you are keeping in your memory, until the session is over"* — confirmed directly by a student and validated.
3. **Then** go to the table **once**, using that bitmap to guide a single, more efficient sweep — rather than bouncing back and forth between index and table per row the way a plain index scan does.

**The core tradeoff, stated directly:** bitmap heap scan is the middle ground between a plain index scan (efficient for few, scattered matches) and a sequential scan (efficient when you'll touch most of the table anyway) — used specifically when the number of matching rows is too large for repeated index-then-table lookups to be efficient, but still selective enough that a full sequential scan would be wasteful.

---

## 5. effective_io_concurrency — Only Matters for Bitmap Scans

**Stated directly and repeated for emphasis: this parameter is relevant *only* during a bitmap heap scan** — it governs how many concurrent I/O requests can be issued while sweeping the table using the bitmap built in step 2 above. Outside of a bitmap heap scan, this parameter has no effect at all.

**Same hardware-driven logic as `random_page_cost`:** hard disk drives can't usefully service many concurrent I/O requests at once (hence a low value like `2`); SSDs can genuinely parallelize far more I/O requests simultaneously (hence `200`). **Directly confirmed: this is exactly why the two parameters are grouped together** — both exist specifically to tell the planner how much your underlying storage can be trusted with random/concurrent access.

---

## 6. max_worker_processes, max_parallel_workers, max_parallel_workers_per_gather — Confirmed Live

A live, incremental demo confirming the exact hierarchy described in the earlier explanation:

```sql
set max_parallel_workers_per_gather = 1;
explain analyze select ... ;
-- Workers Planned: 1, Workers Launched: 1 -- plain, no real parallelism
```
```sql
set max_parallel_workers_per_gather = 2;
-- Workers Planned: 2, Workers Launched: 2
```
```sql
set max_parallel_workers_per_gather = 4;
-- Workers Planned: 4, Workers Launched: 4 (against a machine with enough CPUs/max_worker_processes to supply them)
```

**Then, deliberately constrained from the other end** — reducing `max_worker_processes` (requires restart) down to `3`, while `max_parallel_workers_per_gather` was still `4`:
```
Workers Planned: 4
Workers Launched: 2
```
**Confirmed directly and worked through with the class as a quiz:** "Planned" reflects what the query *wanted* based on `max_parallel_workers_per_gather`; "Launched" reflects what it actually *got*, capped by the smaller of `max_worker_processes` and `max_parallel_workers`. The class initially guessed 3 or 4 launched workers and had to reason through why it landed at 2 — **one worker process is always reserved as the "gather" coordinator itself**, so the actual parallel-scan worker count is `max_worker_processes − 1` in a constrained scenario like this (3 available, 1 reserved for gather, 2 left for scanning).

**Directly summarized:** *"In real time, we will never get any of these four values [exactly as configured] — once we define them based on our CPU count, that's it."* The practical takeaway isn't to memorize the exact arithmetic of worker allocation, but to understand that these four parameters form a capped hierarchy, sized off your actual CPU count, and to move on.

---

## 7. Parallelism Only Applies to SELECT

**A direct, important clarification, in response to a student's question:** parallel query execution — everything governed by `max_parallel_workers_per_gather` and friends — applies **only to `SELECT` statements**, never to `INSERT`/`UPDATE`/`DELETE`. *"Only parallel selects."* DML operations are never parallelized by these settings.

---

## 8. max_parallel_maintenance_workers — Confirmed on CREATE INDEX

```sql
create index idx_name on table_name(column_name);
```
```bash
ps -ef | grep postgres
```
**Confirmed live:** two distinct processes appear during index creation — the main `CREATE INDEX` process, plus a separate "parallel worker" process tied to the same PID — directly demonstrating `max_parallel_maintenance_workers` in action, the maintenance-specific counterpart to the query-level parallel parameters covered above.

---

## 9. huge_pages — Live Troubleshooting

**Setting `huge_pages = on` and attempting to start PostgreSQL without first provisioning OS-level huge pages fails outright:**
```
requested shared memory size ... 
```
**Confirmed directly:** the error message reports roughly how much memory PostgreSQL actually needs in huge-page form — driven by `shared_buffers` plus the rest of shared memory (`wal_buffers`, etc.) combined. With `shared_buffers = 128MB`, the requested amount was around 150MB; raising `shared_buffers` to 900MB pushed the requested amount to roughly 1GB, confirming the request scales directly with configured shared memory.

**The actual fix, demonstrated live:**
```bash
sysctl -w vm.nr_hugepages=100    # allocate 100 huge pages...
```
**Still failed** — because each huge page defaults to **2MB**, so 100 pages = only 200MB, insufficient for the ~1GB PostgreSQL was requesting at that point. Also noted live: **insufficient free system memory** at the time blocked the full allocation from succeeding, requiring a `sync; echo 3 > /proc/sys/vm/drop_caches`-style cache-clear step before more huge pages could actually be granted.

**Once sized correctly** (raising the page count so total huge-page memory exceeded the requested amount), PostgreSQL started successfully using huge pages — confirmed via `SHOW huge_pages;` and checking free/allocated page counts.

**Direct, honest closing note on `huge_pages`:** *"How much benefit you are going to get? It depends."* — not a guaranteed, dramatic win, but a genuine option worth having configured correctly on a production system with a large `shared_buffers`.

---

## 10. The Short List — What You'll Actually Touch Again

**Stated directly and unambiguously, after walking the full PgTune parameter list start to finish:** *"Out of all these values, once you set them in your production, you will never touch [most of them] ... unless you change your computer."*

**The only three parameters realistically revisited after initial setup, named explicitly:**
1. **`max_connections`** — because your actual concurrency need is a moving target you can't perfectly predict up front, and may need raising as real usage patterns emerge.
2. **`work_mem`** — because it's frequently tuned per database or per user for specific workload differences (e.g. a reporting database/user needing more than an OLTP one).
3. **`max_wal_size`** — because WAL volume/checkpoint frequency needs can shift as write activity grows.

**Everything else** — `shared_buffers`, `effective_cache_size`, `random_page_cost`, `effective_io_concurrency`, `huge_pages`, the parallel-worker parameters, etc. — is set once, correctly, based on the machine's actual hardware specs at provisioning time, and left alone unless the underlying hardware itself changes (e.g. moving to a bigger machine, at which point you simply resize proportionally and move on).

---

## 11. pgconfig.org — an Alternative to PgTune

Introduced directly as a second option alongside PgTune, with a specific stated advantage: *"what advantage you get from pgconfig.org is it will try to help you understand why you need to set that value, and what happens if you set that value"* — i.e., it pairs each recommended value with an explanation, rather than just outputting a number the way PgTune does. **Explicitly confirmed as equally valid for real production use** — not a toy tool, the same category of resource as PgTune, just with more built-in explanation.

---

## 12. Questions Discussed in This Session

**Q1. If we're using all available CPU capacity for PostgreSQL parallel operations, doesn't that hurt performance for other processes on the machine?**

Directly addressed with a worked example: CPUs aren't pre-reserved or dedicated to PostgreSQL just because `max_worker_processes` is set to the total CPU count — they're used *on demand*, only while an actual operation needs them. If something else on the machine (e.g. antivirus scanning, unrelated cron jobs) needs a CPU at the same moment, it competes for the same pool the normal way any OS process would — `max_worker_processes` isn't a reservation, it's a ceiling on how many PostgreSQL is *allowed* to use if available.

---

**Q2. Does the "Workers Planned vs. Workers Launched" gap mean the query silently runs slower or fails?**

No — it simply runs with fewer parallel workers than it ideally wanted, gracefully. The query still completes correctly; it just gets less parallelism than the `Planned` figure suggests, capped by whatever `max_worker_processes`/`max_parallel_workers` can actually supply at that moment (including contention from other concurrently-running parallel queries).

---

**Q3. Is there a way to track *who* changed a configuration parameter and when, for audit purposes?**

No — confirmed directly: PostgreSQL has no built-in mechanism to audit *who* modified a configuration value. The alert log can show *when* a reload/restart picked up a new value, but not the identity of whoever made the edit — that has to be handled through external change-management process (e.g. version-controlling `postgresql.conf`, requiring comments/tickets for changes), not anything PostgreSQL itself tracks.

---

**Q4. Does effective_io_concurrency affect anything outside of a bitmap heap scan, like a plain sequential or index scan?**

No — confirmed directly and repeated: this parameter's effect is scoped entirely to bitmap heap scans. It has no bearing on sequential scans or plain index scans at all.
