# psql, .pgpass, .psqlrc & pgbench — Reference Guide

A full walkthrough of `psql` itself — connection defaults, meta-commands, the `-c`/`-f`/`-i` flags, client/server version mismatches — followed by passwordless connectivity via `.pgpass`, customizing the shell via `.psqlrc`, and closing with `pgbench` used as a real capacity-planning tool (CPU saturation demonstrated live via `top`).

---
## Table of Contents

- [1. Every Tool Has --help](#1-every-tool-has---help)
- [2. Connection Defaults](#2-connection-defaults)
- [3. Superuser # vs. Normal User >](#3-superuser--vs-normal-user-)
- [4. Meta-Commands — \? and \h](#4-meta-commands---and-h)
- [5. Switching Databases and Users Mid-Session — \c](#5-switching-databases-and-users-mid-session--c)
- [6. -c and -f — Running SQL from the OS Shell](#6--c-and--f--running-sql-from-the-os-shell)
- [7. Client/Server Version Mismatch, Revisited](#7-clientserver-version-mismatch-revisited)
- [8. Environment Variables — \set and PGHOME-Style Export](#8-environment-variables--set-and-pghome-style-export)
- [9. .pgpass — Passwordless Connections](#9-pgpass--passwordless-connections)
- [10. .psqlrc — Customizing the Shell](#10-psqlrc--customizing-the-shell)
- [11. Both Files Are Local-Machine Only](#11-both-files-are-local-machine-only)
- [12. Skipping .psqlrc — psql -X](#12-skipping-psqlrc--psql--x)
- [13. pgbench — Initializing and Scaling](#13-pgbench--initializing-and-scaling)
- [14. pgbench as a Real Capacity-Planning Tool](#14-pgbench-as-a-real-capacity-planning-tool)
- [15. CPU Scheduling, Explained via a Desktop Analogy](#15-cpu-scheduling-explained-via-a-desktop-analogy)
- [16. The Business Question TPS Actually Answers](#16-the-business-question-tps-actually-answers)
- [17. Questions Discussed in This Session](#17-questions-discussed-in-this-session)

---

## 1. Every Tool Has --help

```bash
cd /usr/pgsql-18/bin
psql --help
pg_dump --help
```
**Confirmed directly, as a general pattern worth internalizing:** nearly every PostgreSQL command-line tool's `--help` output is organized into the same three sections — **general options**, **tool-specific options**, and **connection options**. Once you recognize this shape, `--help` on any unfamiliar tool (`pgbench`, `pg_restore`, whatever) becomes immediately navigable, rather than needing to memorize each tool's flags individually.

**Explicit, repeated advice on using `--help` as a study habit:** *"go through them once... you don't need to remember everything, but at least try to go through each and every option"* — the goal isn't memorization, it's knowing an option exists so you can find it again later when you actually need it (e.g. `-H` for HTML-formatted output, mentioned directly as an example of an easy-to-miss option).

---

## 2. Connection Defaults

```bash
/usr/pgsql-18/bin/psql -U postgres -d postgres -p 5432 -h 192.168.108.131
```

**The five things any connection needs:** username, password, database name, port, host/IP address.

```bash
psql
```
**Run with no options at all, `psql` falls back to defaults for every unspecified piece:** host = current machine (local connection), port = `5432`, username = `postgres`, database name = `postgres`. **Confirmed directly:** a local connection (no `-h` given, or `-h` pointing at the local machine) does **not** prompt for a password at all — only remote connections do, consistent with `pg_hba.conf`'s typical `trust`/`peer` handling of local Unix-socket connections.

**Confirmed directly, addressing a direct question:** flag order doesn't matter (`-h` before `-U` or after both work identically) — what matters is that each flag is followed by the correct type of value (`-U` always takes a username, never a database name).

---

## 3. Superuser # vs. Normal User >

**Confirmed directly and demonstrated live:** the `psql` prompt symbol itself tells you your privilege level — `postgres=#` (a `#` after the database name) means you're connected as a **superuser**; `adb=>` (a `>` instead) means you're connected as a **normal user**. This is a quick, at-a-glance way to confirm privilege level without running any query.

---

## 4. Meta-Commands — \? and \h

```
\?
```
Lists every **meta-command** (backslash command) `psql` supports — described directly as *"help commands"* for the client itself. Demonstrated live finding `\di` (list indexes), `\dm` (list materialized views), `\dt+` (tables with extra detail like size), among others.

**Explicit, repeated guidance on memorization:** *"we hardly remember 3 to 4 things... the rest of the shortcodes, you don't need to remember"* — the expectation is knowing `\?` exists and being comfortable searching it live, not having every meta-command memorized.

```
\h create table
\h create user
\h alter table
```
A **separate** help system from `\?` — `\h` gives the actual **SQL syntax** for a given statement, not a `psql` client command. Demonstrated directly: not remembering the exact `CREATE USER` syntax (with options like `NOLOGIN`) is fine — `\h create user` prints it directly, with all valid options.

---

## 5. Switching Databases and Users Mid-Session — \c

```
\c adb
```
Switches to a different database, **keeping the current user**.

```
\c adb auser
```
Switches to a different database **and** a different user in one command — confirmed the argument order is always **database name first, then username** (attempting the reverse order fails with a role-does-not-exist error, demonstrated live as a deliberate mistake).

```
\c - auser
```
The `-` placeholder means **keep the current database**, only change the user.

**Direct clarification of a common point of confusion, addressed head-on:** `\c` (inside `psql`, switching database/user mid-session) and `-c` (an OS-shell flag, covered next) are **completely different** despite the similar-looking name — one is a meta-command for an already-connected session, the other is a shell flag for running a single command without staying connected at all.

---

## 6. -c and -f — Running SQL from the OS Shell

```bash
psql -c "select * from employee;"
```
**`-c`** — run a **single command** directly from the OS shell, without ever landing in an interactive `psql` prompt.

```bash
psql -c "select * from employee;" -c "select count(1) from emp;"
```
**Confirmed live: multiple `-c` flags can be chained in one call**, each executed in sequence, with output for each printed in order.

```bash
vi insert.sql
# insert into employee values (1, 100);
psql -f insert.sql
```
**`-f`** — execute an entire **SQL script file** from the OS shell.

**The distinction stated directly:** *"if you want to execute a SQL file, you use `-f`. If you want to execute a command, you use `-c`."*

```
\i insert.sql
```
**A third, related option — `\i` (lowercase) — runs a script file from *inside* an already-open `psql` session**, without needing to drop back to the OS shell first. Distinguished directly from `-f` (an OS-shell flag) precisely because it's a meta-command usable mid-session.

---

## 7. Client/Server Version Mismatch, Revisited

**A real, live-encountered problem:** running `psql` on a machine with both v18 and v19 installed connected using the **v19 client** by default (confirmed via the connection banner: `psql (19beta1, server 18.4)`).

```bash
export PATH=/usr/pgsql-18/bin:$PATH
```
**The fix, matching the earlier `PGHOME` pattern from an prior session:** explicitly prepend the desired version's `bin` directory to `PATH` (typically placed in `.bash_profile` for persistence) so the correct client binary is picked up by default, rather than whichever version happens to appear first in the existing `PATH`.

---

## 8. Environment Variables — \set and PGHOME-Style Export

```
\set
```
Lists `psql`'s current default environment/session variables — confirmed showing things like `AUTOCOMMIT = on`, `DBNAME = postgres`, and similar defaults.

```bash
export PGDATABASE=adb
export PGPASSWORD=...
export PGPORT=5433
```
**Confirmed directly:** these client-side defaults can be overridden by exporting the corresponding environment variable (in `.bash_profile` or the current shell) — e.g. wanting a non-default port as your personal default going forward, without needing to type `-p` on every single invocation.

---

## 9. .pgpass — Passwordless Connections

```
hostname:port:database:username:password
```
```bash
cat ~/.pgpass
```
```
*:*:*:postgres:postgres
```
`*` acts as a wildcard for any field — this example matches any host, any port, any database, specifically for the `postgres` user.

```bash
chmod 600 /var/lib/pgsql/.pgpass
```
**Confirmed as a hard requirement, not optional hardening:** the file's permissions must be restricted to the owner only (`600`) — PostgreSQL's client tools will silently refuse to use a `.pgpass` file with looser permissions, treating it as a security risk.

**The direct, practical use case named:** avoiding password prompts for automated activity — **cron jobs** and **replication** were both named directly as the reason this matters in real production use, since neither can interactively type a password when triggered.

---

## 10. .psqlrc — Customizing the Shell

```
~/.psqlrc
```
A per-user file, read automatically every time that user starts `psql`, letting you customize the prompt, define shortcuts, and run startup commands.

```
\set PROMPT1 '%n@%m:%>%/%R%# '
```
Demonstrated live: customizing the connection prompt to show host, port, username, and database together, rather than the terse default.

```
\echo 'Welcome to PostgreSQL! \n'
\echo 'Type :version to see the PostgreSQL version. \n'
\set version 'SELECT version();'
\echo 'Type :locks to see locks information. \n'
\set locks 'select * from pg_locks;'
```

**Confirmed live, and worked through as a mini-quiz on the difference between two similar-looking lines:** `\set version '...'` defines a **named shortcut**, invoked later with `:version` — it does not run the query immediately at login. A separate line assigning a query's *result* to a variable (e.g. capturing a count into a variable, then displaying it with `:variable_name`) behaves differently — one stores the query text itself to be run on demand, the other stores a value already computed.

**Practical framing given directly:** this is a place to park your **frequently-used DBA queries** as short, memorable aliases — locks, wait events, wraparound-check queries were all named directly as good candidates. **An explicit alternative offered:** rather than cramming every query into `.psqlrc`, you can equally keep a personal folder of `.sql` files (e.g. `~/dumps/locks.sql`, `~/dumps/wait_events.sql`) and run them with `-f` or `\i` as needed — **there's no single correct approach, it's a matter of personal workflow preference.**

---

## 11. Both Files Are Local-Machine Only

**Confirmed directly, addressing a student's question:** both `.pgpass` and `.psqlrc` are **local-machine, local-user-only files** — they have no effect for a remote client connecting *to* this machine, and must live specifically in that OS user's **home directory** (not any other location) to be picked up at all.

---

## 12. Skipping .psqlrc — psql -X

```bash
psql -X
```
**Confirmed live:** connects **without** reading `.psqlrc` at all — useful when you specifically want the plain, unmodified default `psql` behavior for a given session, without your usual customizations (echoed messages, custom prompt, shortcuts) interfering.

---

## 13. pgbench — Initializing and Scaling

```bash
pgbench --help
```
Confirmed described directly in its own help text as *"a benchmarking tool for PostgreSQL."*

```bash
pgbench -i postgres
```
Creates the 4 standard `pgbench_*` tables. **Confirmed live: `pgbench_accounts` alone came out to 13MB** at the default scale factor.

```bash
pgbench -i -s 10 postgres
```
**`-s 10`** — scale factor 10, confirmed live to produce a `pgbench_accounts` table roughly **10× larger** (130MB) than the default — directly validating that scale factor is a straightforward linear multiplier on the default dataset size.

```bash
pgbench -i -s 5 postgres
```
Confirmed live producing the proportionally smaller **~64MB** result — reinforcing the same linear relationship at a different multiplier.

---

## 14. pgbench as a Real Capacity-Planning Tool

```bash
pgbench -c 10 -T 30 postgres
```
**`-c 10`** — 10 concurrent simulated clients. **`-T 30`** — run for 30 seconds. Cross-checked live against `pg_stat_activity` in a second session, confirming real, concurrent sessions actually appear — some `active`, some `idle`, some `idle in transaction` — while the benchmark runs, tying directly back to the session-state material from the earlier connection-establishment session.

```bash
pgbench -c 10 -T 30 -P 3 postgres
```
**`-P 3`** — print progress every 3 seconds instead of waiting silently for the full 30-second run to finish, surfacing a live-updating **TPS (transactions per second)** figure as the run progresses. **Confirmed live: TPS was explicitly flagged as a common, important interview question** — *"go check your TPS in your environment, whatever it is — Postgres, Oracle, MySQL, SQL Server."*

**The live stress-test demo, using `top` alongside `pgbench`:**
```bash
top
```
Baseline: system essentially idle (~99% CPU idle). Increasing `pgbench`'s concurrent client count in successive runs (`-c 10`, `-c 20`, `-c 50`, ...) was tracked directly against the `top` idle percentage dropping correspondingly (confirmed live: 99% → ~57–58% → ~26% → lower still as concurrency increased further) — **a live, visible demonstration that pushing more concurrent load directly consumes more CPU capacity, with a real, observable ceiling.**

**Explicitly confirmed: `pgbench`'s `-c` (client count) is capped by the server's own `max_connections`** — attempting `-c 120` against an instance with `max_connections = 100` fails outright; this is exactly why the demo was deliberately kept at 90 concurrent clients against a 100-connection ceiling.

---

## 15. CPU Scheduling, Explained via a Desktop Analogy

**Built directly to explain why performance degrades gradually rather than failing abruptly once concurrency exceeds CPU count:** on a 4-core machine, running four demanding tasks (editing a document, a background copy operation, playing music, running a video player) means each genuinely gets its own core — no contention. **Adding a fifth concurrent task doesn't fail** — it gets CPU time too, but now **all five** tasks share time on four cores via **context switching**: the OS gives each task a small slice of CPU time (*"X amount of work"*), then switches to the next, cycling continuously. **This is the direct, tangible explanation for the everyday experience of typing into a word processor and seeing the character appear with a slight, noticeable delay under heavy system load** — the CPU genuinely was busy with something else in that instant.

**Directly generalized back to the database context:** exactly the same mechanism explains why database performance degrades smoothly (not catastrophically) as concurrent session count climbs past the machine's core count — every additional concurrent query genuinely gets served, just with proportionally less CPU time per query as contention increases.

---

## 16. The Business Question TPS Actually Answers

**Framed directly as the real-world payoff of running this kind of benchmark, tying it to an actual business scenario:** *"1,500 users will log in and purchase something. What will happen? Based on your benchmarking test, only 1,400 can perform at a time. The other 100 has to suffer — they will get [the] next cycle of CPU."*

**The resulting decision framework, stated directly as the two available levers:** once you know your system's actual measured capacity via benchmarking, the choice is between **(a) tuning the query/workload itself** to reduce its CPU cost, or **(b) increasing the underlying compute** to raise the ceiling — *"there is always a difference that we need to identify"* between "this is genuinely the best this query can do" and "this system needs more CPU." Benchmarking with `pgbench` is presented as the concrete, measurable way to know which side of that line you're actually on, rather than guessing.

---

## 17. Questions Discussed in This Session

**Q1. Is `-c` inside `psql` (switching connections) the same thing as `-c` used from the OS shell?**

No — confirmed directly and explicitly distinguished: `\c` (backslash, inside an active `psql` session) switches the current session's database/user. `-c` (hyphen, an OS-shell flag to `psql` itself) runs one single SQL command non-interactively and doesn't require ever entering an interactive session at all. The similar naming is coincidental, not a sign of shared behavior.

---

**Q2. Does `pgbench`'s `-c` (client count) relate to the server's `max_connections` setting?**

Yes — confirmed directly: `pgbench -c` simulates that many concurrent client connections, and is bounded by whatever `max_connections` the target server actually allows. Attempting to request more concurrent `pgbench` clients than `max_connections` permits fails outright, which is exactly why the live demo was deliberately capped below the configured limit.

---

**Q3. Are `.pgpass` and `.psqlrc` usable for connections initiated from a remote machine, or only locally?**

Only locally — confirmed directly: both files are read from the connecting OS user's own home directory on the machine `psql` is actually being run from. A remote client connecting to this server has no access to, and is unaffected by, this server's own `.pgpass`/`.psqlrc` files.

---

**Q4. If a `.psqlrc` shortcut is defined with `\set name 'some query'`, does it run immediately when `psql` starts, or only when invoked?**

Only when invoked — confirmed directly and demonstrated live via a comparison of two similar-looking `.psqlrc` lines: `\set` merely *registers* the shortcut text; it doesn't execute anything at login. The shortcut only actually runs when you reference it later in the session using the `:name` syntax.
