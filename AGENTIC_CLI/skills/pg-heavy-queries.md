---
name:        pg-heavy-queries
description: Rank the heaviest Postgres queries via pg_stat_statements — by total time, by disk I/O, by user.
version:     1
trigger_keywords: [postgres, slow query, pg_stat_statements, performance, hit ratio, top consumers]
tools:       [bash]
requires:    [POSTGRES_RUNBOOK]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "the database is slow", "find the most expensive queries", "which service is hammering the DB". Use when the symptom is **sustained cost/load** and you need to know where time goes.

Not a use case: queries hung on a lock (use `pg-blocked-sessions`); a one-off slow query already identified — go straight to `EXPLAIN ANALYZE`.

## Prerequisites

- The `pg_stat_statements` extension installed and enabled (`shared_preload_libraries`).
- `psql` access to a role that can read it (superuser or `pg_monitor`).

## Procedure

1. **Top queries by total execution time** — where wall-clock time actually goes:
   ```sql
   SELECT query, calls,
          total_exec_time AS total_ms,
          mean_exec_time AS avg_ms,
          rows,
          round((total_exec_time / calls)::numeric, 2) AS avg_time_per_call_ms
   FROM pg_stat_statements
   ORDER BY total_exec_time DESC
   LIMIT 10;
   ```
   High `total_exec_time` = the biggest aggregate cost (often frequent + expensive). High `avg_time_per_call_ms` = individually slow. Cross with `calls` to tell frequency from complexity.
2. **Top queries by disk I/O** — missing-index candidates:
   ```sql
   SELECT query, shared_blks_read, shared_blks_hit,
          round(100 * shared_blks_hit::numeric / nullif(shared_blks_hit + shared_blks_read, 0), 2) AS hit_ratio
   FROM pg_stat_statements
   ORDER BY shared_blks_read DESC
   LIMIT 10;
   ```
   High `shared_blks_read` = many disk reads (possible missing index). `hit_ratio` below ~90% = inefficient caching.
3. **Ranking by user** — which account/service drives the load:
   ```sql
   SELECT rolname AS user,
          sum(total_exec_time) AS total_exec_time_ms,
          sum(calls) AS total_calls,
          round((sum(total_exec_time)/sum(calls))::numeric, 2) AS avg_time_per_call_ms
   FROM pg_stat_statements s
   JOIN pg_roles r ON r.oid = s.userid
   GROUP BY rolname
   ORDER BY total_exec_time_ms DESC
   LIMIT 10;
   ```
4. **Drill down.** Take the worst offender and run `EXPLAIN (ANALYZE, BUFFERS)` on it to confirm the plan and the fix (index, rewrite, batching).
5. **Reset for a clean window** when running a controlled test or measuring after a deploy:
   ```sql
   SELECT pg_stat_statements_reset();
   ```

## Verification

- After applying a fix, re-run step 1: the offending query dropped in `total_ms` or left the top 10.
- For an index fix, step 2 shows a higher `hit_ratio` / lower `shared_blks_read` for that query.

## Anti-cases

- Stats accumulated over months → a one-off heavy job dominates the ranking without being a real problem; reset (step 5) and observe a representative window.
- The top query is cheap per call but called millions of times → the fix is in the caller (caching, N+1), not the SQL.
