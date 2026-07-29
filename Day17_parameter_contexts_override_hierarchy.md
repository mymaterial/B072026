# Parameter Contexts, Reload vs. Restart, and the 7-Level Override Hierarchy

How to actually determine whether a parameter change needs a restart or just a reload, and the full seven-level hierarchy PostgreSQL uses to resolve a parameter's effective value — from `postgresql.conf` all the way down to a single transaction — all demonstrated live with `work_mem` at every level.

---
## Table of Contents

- [1. pg_settings and the context Column](#1-pg_settings-and-the-context-column)
- [2. The Seven Context Values, Decoded](#2-the-seven-context-values-decoded)
- [3. Two Ways to Check What a Parameter Needs](#3-two-ways-to-check-what-a-parameter-needs)
- [4. Comment Means Default, Not Disabled](#4-comment-means-default-not-disabled)
- [5. reload vs. restart, Defined Plainly](#5-reload-vs-restart-defined-plainly)
- [6. Level 1 — postgresql.conf, Top to Bottom](#6-level-1--postgresqlconf-top-to-bottom)
- [7. Level 2 — postgresql.auto.conf via ALTER SYSTEM](#7-level-2--postgresqlautoconf-via-alter-system)
- [8. Why ALTER SYSTEM Writes to a Separate File](#8-why-alter-system-writes-to-a-separate-file)
- [9. Levels 3–5 — Database, User, and the Combination](#9-levels-35--database-user-and-the-combination)
- [10. Where Levels 3–5 Actually Live — pg_db_role_setting](#10-where-levels-35-actually-live--pg_db_role_setting)
- [11. Levels 6–7 — Session and Transaction](#11-levels-67--session-and-transaction)
- [12. The Full Precedence Order](#12-the-full-precedence-order)
- [13. Resetting a Value at Any Level](#13-resetting-a-value-at-any-level)
- [14. When to Actually Use Each Level](#14-when-to-actually-use-each-level)
- [15. Questions Discussed in This Session](#15-questions-discussed-in-this-session)

---

## 1. pg_settings and the context Column

```sql
select distinct context from pg_settings;
```
```
      context
-------------------
 postmaster
 superuser-backend
 user
 internal
 backend
 sighup
 superuser
(7 rows)
```

Every parameter in PostgreSQL carries a `context` value in this catalog view — and that single column tells you everything you need to know about how a given parameter can legally be changed, without needing to memorize anything per-parameter.

---

## 2. The Seven Context Values, Decoded

| Context | What it means |
|---|---|
| `postmaster` | Requires a full **restart** to take effect |
| `sighup` | **Reload** is sufficient — but cannot be set at user, database, or session level; instance-wide only |
| `superuser` | Reload is fine; can be set at user/database level, but only by a superuser |
| `superuser-backend` | Reload is fine, superuser-scoped |
| `user` | Reload is fine, **and** can be set at user level, database level, or session level — the most flexible category |
| `backend` | Reload is fine |
| `internal` | Fixed at compile time — not meant to be changed by an administrator at all |

**The one distinction that matters most in daily practice, stated directly:** `sighup` parameters need only a reload, but **cannot** be scoped to a specific user/database/session — they're always instance-wide. `user`-context parameters get the full flexibility — any of the seven levels covered later in this document.

---

## 3. Two Ways to Check What a Parameter Needs

**Method 1 — query `pg_settings` directly:**
```sql
select name, setting, context
from pg_settings
where name in ('shared_buffers', 'max_parallel_workers_per_gather', 'min_wal_size', 'work_mem');
```
Confirmed live: `shared_buffers` → `postmaster` (restart); `max_parallel_workers_per_gather`, `work_mem` → `user` (reload, any scope); `min_wal_size` → `sighup` (reload, but instance-wide only — cannot be set at user level).

**Method 2 — read `postgresql.conf` directly.** Every parameter that needs a restart has this noted in an inline comment right next to it in the shipped configuration file (`# (change requires restart)`). **What this second method can't tell you:** whether a given parameter can be scoped to a user/database/session — for that, you still need `pg_settings`.

---

## 4. Comment Means Default, Not Disabled

**A recurring, deliberately reinforced point:** a commented-out line in `postgresql.conf` (e.g. `#log_checkpoints = on`) means *"use the default value"* — which, for that specific example, is already `on`. **It does not mean "disabled."** To genuinely turn something off, you have to uncomment it and explicitly set it to `off` — simply removing the `#` in front of an already-defaulting-to-on parameter changes nothing.

```ini
# log_disconnections = off    -- commented; default applies (off, in this case)
log_disconnections = on       -- explicitly enabled
```
**To enable `log_disconnections`, two things are required together:** uncomment the line, **and** set it to `on` — then reload.

---

## 5. reload vs. restart, Defined Plainly

Stated directly, in response to a student's question about the difference:
- **Reload** — *"you are just re-reading your configuration file and applying the new configuration values to the upcoming sessions."* The running instance keeps running throughout; no downtime.
- **Restart** — *"restarting your server."* The instance stops and starts again — genuine, if brief, downtime.

```sql
select pg_reload_conf();
```
or, from the OS shell:
```bash
pg_ctl reload -D /u01/pgsql/18
```
Both accomplish the same reload — confirmed as functionally interchangeable, one from inside `psql`, one from the OS prompt.

---

## 6. Level 1 — postgresql.conf, Top to Bottom

```ini
work_mem = 8MB

#------------------------------------------------------------------------------
# CUSTOMIZED OPTIONS
#------------------------------------------------------------------------------
work_mem = '12MB'   ## changed on 21st July
work_mem = '16MB'   ## changed on 22nd July
```

**Confirmed live and explained directly:** the postmaster reads `postgresql.conf` **top to bottom**. If the same parameter appears more than once (as deliberately demonstrated here, with three separate `work_mem` lines), **the last one read wins** — after a reload, `work_mem` resolved to `16MB`, the final occurrence in the file, not the first.

**Practical implication, stated directly:** you're not restricted to editing a parameter only where it originally appears in the file — appending a new value further down (e.g. in a dedicated "customized options" section at the bottom, a common convention) works exactly the same as editing it in place, as long as it's read after the earlier occurrence.

---

## 7. Level 2 — postgresql.auto.conf via ALTER SYSTEM

```sql
alter system set work_mem = '24MB';
```
```bash
cat /u01/pgsql/18/postgresql.auto.conf
```
```
# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
work_mem = '24MB'
```

**Confirmed directly:** `ALTER SYSTEM` doesn't touch `postgresql.conf` at all — it writes to a **separate** file, `postgresql.auto.conf`, which is read *after* `postgresql.conf` (meaning its values win over whatever's in the main file, for the same reason later-read values win within a single file). Still requires a reload to take effect — `ALTER SYSTEM` alone doesn't apply the change immediately.

**Confirmed live:** running `ALTER SYSTEM SET work_mem = ...` a second time with a different value **overwrites** the existing entry in `postgresql.auto.conf` — it doesn't append a second line the way manually editing `postgresql.conf` can; there's no history retained of prior `ALTER SYSTEM` values.

---

## 8. Why ALTER SYSTEM Writes to a Separate File

**A direct question worked through:** why does `ALTER SYSTEM` write to a different file instead of just editing `postgresql.conf` directly?

**Answer, stated plainly:** *"you don't want to edit this file [directly]."* `postgresql.auto.conf` exists specifically so that programmatic changes (via SQL, from inside the database) never have to parse, modify, and rewrite the main configuration file — a much riskier operation than appending/overwriting a small, dedicated, machine-managed file.

**A real, practical tradeoff flagged directly:** manually editing `postgresql.conf` lets you leave a comment recording *why* and *when* a value changed (as shown in Section 6's `## changed on 21st July` example) — `ALTER SYSTEM` has **no such mechanism**; it silently overwrites the prior value with no trace of what it used to be or when it changed. **Some organizations mandate `ALTER SYSTEM` exclusively** (no manual `postgresql.conf` editing at all) for consistency/tooling reasons — there's no universal right answer, it's a project-level policy decision, with a real traceability cost either way depending on how disciplined manual edits are (an undocumented manual edit is just as untraceable as an `ALTER SYSTEM` change).

---

## 9. Levels 3–5 — Database, User, and the Combination

```sql
alter database adb set work_mem = '32MB';
```
```sql
alter user auser set work_mem = '48MB';
```
```sql
alter user postgres in database adb set work_mem = '64MB';
```

**Confirmed directly: unlike Levels 1–2, these three take effect immediately on the *next* connection — no reload required at all.** *"If you are changing at user level, session level, and database level, you don't need to reload. Just, you need to reconnect."*

**The distinction between `ALTER DATABASE` and `ALTER USER`, asked directly and answered plainly:** `ALTER DATABASE` changes the value for anyone connecting to that specific database, regardless of who they are; `ALTER USER` changes it for that specific user, regardless of which database they connect to. The **combination form** (`ALTER USER ... IN DATABASE ...`) is more specific than either alone — it only applies when that exact user connects to that exact database.

---

## 10. Where Levels 3–5 Actually Live — pg_db_role_setting

```sql
select * from pg_db_role_setting;
```
```
 setdatabase | setrole |    setconfig
-------------+---------+-----------------
       24647 |       0 | {work_mem=32MB}
           0 |   24646 | {work_mem=48MB}
       24647 |      10 | {work_mem=64MB}
```

**Confirmed directly:** database-level, user-level, and combination-level settings are **not** written to `postgresql.conf` or `postgresql.auto.conf` at all — they live in this dedicated system catalog, `pg_db_role_setting`, keyed by **object ID (OID)**, not by name.

```sql
select oid, datname from pg_database;
```
Reading the table: a row with `setdatabase = 24647, setrole = 0` is a **pure database-level setting** (no specific role, `0` meaning "any role") — applies to `adb` (OID 24647) regardless of who connects. A row with `setdatabase = 0, setrole = 24646` is a **pure user-level setting** — applies to that user (`auser`, OID 24646) regardless of which database they connect to. A row with both non-zero is the **combination** — only applies when that exact user connects to that exact database.

**Confirmed directly: PostgreSQL, as an object-relational database, assigns a numeric OID to essentially everything** — databases, users, schemas, tables, indexes — and `pg_db_role_setting` is a clean, concrete example of that OID-based identification in action, rather than storing raw names.

---

## 11. Levels 6–7 — Session and Transaction

```sql
set work_mem = '128MB';
show work_mem;   -- 128MB
```
```sql
begin;
set work_mem = '1GB';
show work_mem;   -- 1GB, but ONLY within this transaction
```

**Session level (`SET`)** overrides every level below it (config file, `auto.conf`, database, user, combination) for the remaining duration of that one connection — gone the moment you disconnect.

**Transaction level (`SET` inside `BEGIN`)** goes one step further — the override only holds for the duration of that specific transaction block; once it commits or rolls back, the session reverts to whatever its own session-level (or lower) value was.

---

## 12. The Full Precedence Order

Confirmed directly, and consistent with the numbering used throughout the session:

```
1. postgresql.conf          (last matching line in the file wins)
2. postgresql.auto.conf     (ALTER SYSTEM)
3. Database level            (ALTER DATABASE ... SET ...)
4. User level                 (ALTER USER ... SET ...)
5. Combination (user + database)  (ALTER USER ... IN DATABASE ... SET ...)
6. Session level              (SET ...)
7. Transaction level          (SET ... inside BEGIN)
```

**Stated directly as the core rule: each level overrides everything above it, in order** — *"it will start overriding all these values as it goes forward until 7... whatever is set at session level, or the transaction level, that will be the final value."* This is exactly the mechanism behind the diagnostic scenario posed directly to the class: *"I logged in, ran `SHOW work_mem`, and got 128MB — but my `postgresql.conf` and `auto.conf` both say 32MB. Where's 128MB coming from?"* — the answer being that some higher-precedence level (database, user, combination, or session) must be setting it, and `pg_db_role_setting` (Section 10) or simply checking each level in order is how you'd actually track it down.

---

## 13. Resetting a Value at Any Level

```sql
alter user auser reset work_mem;
```
**Confirmed live:** this removes the corresponding row from `pg_db_role_setting` entirely — the user-level override is gone, and resolution falls back through to whatever the next level down provides.

```sql
alter system reset max_wal_size;
```
**Confirmed live:** removes that specific line from `postgresql.auto.conf`, leaving any other `ALTER SYSTEM`-set values untouched.

```sql
alter system reset all;
```
Clears **every** `ALTER SYSTEM`-set value from `postgresql.auto.conf` in one command — a full reset back to `postgresql.conf`'s own values.

**All reset operations still require a reload** (for Levels 1–2) or simply take effect on next connection (for Levels 3–5) — resetting isn't special in that regard, it follows exactly the same activation rule as setting a value in the first place.

---

## 14. When to Actually Use Each Level

**Two concrete, realistic use cases given directly, tying the whole hierarchy back to a practical reason to care:**
1. **Database-level `work_mem`:** one database in a multi-database instance runs heavy reporting queries with lots of `ORDER BY`/aggregation — rather than raising `work_mem` instance-wide (Level 1/2), scope it to just that database (Level 3).
2. **User-level `work_mem`:** a dedicated reporting user needs more sort/hash memory than your regular OLTP application user — set it on that specific user (Level 4), leave the OLTP user at the default.

**Direct, blunt framing on how often the instance-wide levels (1–2) actually get touched post-setup:** *"most probably, we will not do these things in configuration file... because why we reset when we set it?"* — reinforcing the same point from the earlier session that only a handful of parameters (`max_connections`, `work_mem`, `max_wal_size`) get revisited at all in a mature production system, and even then, the *scoped* levels (database/user) are frequently the more surgical, appropriate tool rather than a blanket instance-wide change.

---

## 15. Questions Discussed in This Session

**Q1. If a parameter's context is `sighup`, can it still be set at the database or user level?**

No — confirmed directly: `sighup`-context parameters only support a reload (no restart needed), but they are **instance-wide only**. They cannot be scoped to a specific database, user, or session — that flexibility is reserved for `user`-context parameters specifically. `min_wal_size` was used as the concrete example.

---

**Q2. Does changing a parameter at the OS level affect every database under that instance?**

The question was corrected in terms rather than answered as asked: there's no such thing as "the OS level" for this purpose — it's the **configuration file level**, not an OS-level setting. A `postgresql.conf`/`postgresql.auto.conf` change (Levels 1–2) does apply instance-wide, across every database in that cluster, but that's a configuration-file-level scope, not an operating-system-level one.

---

**Q3. Is there any relationship between `postgresql.conf` and `postgresql.auto.conf` beyond one being read after the other?**

No — confirmed directly: `postgresql.auto.conf` isn't linked to or derived from `postgresql.conf` in any deeper way. It's simply a second file, read after the first, that `ALTER SYSTEM` writes to instead of editing the main file directly — the "relationship" begins and ends with read order.

---

**Q4. When resolving the final value of a parameter with settings at multiple levels, does PostgreSQL check all seven levels every time, or stop at the first match?**

Framed directly as a strict override chain, not a "first match wins" search: PostgreSQL applies Level 1, then checks whether Level 2 overrides it, then Level 3, and so on, sequentially through Level 7 — each subsequent level that has a value set overrides everything before it. The final effective value is whatever the **highest-numbered level with an active setting** provides, not the first one found.
