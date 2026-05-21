---
name:        pg-blocked-sessions
description: Inspect active Postgres sessions, identify the blocked/blocking pair, and terminate the blocker with judgment.
version:     1
trigger_keywords: [postgres, lock, blocking, idle in transaction, pg_stat_activity, deadlock]
tools:       [bash]
requires:    [POSTGRES_RUNBOOK]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "queries are stuck", "the database froze", "a lock is holding everything". Use when the symptom is **waiting/contention** — hung requests, transaction timeouts.

Not a use case: a single slow query with no contention (use `pg-heavy-queries`); the database simply under high load with no blocking (same).

## Prerequisites

- `psql` access to a role that can see `pg_stat_activity` (superuser or `pg_monitor`).
- For `pg_terminate_backend`: privilege to terminate the target backend.

## Procedure

1. **Active sessions** — what is running right now:
   ```sql
   SELECT pid, usename, datname, state,
          wait_event_type, wait_event,
          query_start, now() - query_start AS execution_time, query
   FROM pg_stat_activity
   WHERE state <> 'idle'
   ORDER BY execution_time DESC;
   ```
   Signals: high `execution_time` = long-running query; `wait_event_type = 'Lock'` = waiting on a lock; `state = 'idle in transaction'` = a forgotten open transaction (classic lock source).
2. **Blocked/blocking pair:**
   ```sql
   SELECT blocked.pid AS blocked_pid, blocked.usename AS blocked_user, blocked.query AS blocked_query,
          blocking.pid AS blocking_pid, blocking.usename AS blocking_user, blocking.query AS blocking_query
   FROM pg_locks bl
   JOIN pg_stat_activity blocked ON blocked.pid = bl.pid
   JOIN pg_locks bl2 ON bl.locktype = bl2.locktype
                     AND bl.database IS NOT DISTINCT FROM bl2.database
                     AND bl.relation IS NOT DISTINCT FROM bl2.relation
                     AND bl.page IS NOT DISTINCT FROM bl2.page
                     AND bl.tuple IS NOT DISTINCT FROM bl2.tuple
                     AND bl.virtualxid IS NOT DISTINCT FROM bl2.virtualxid
                     AND bl.transactionid IS NOT DISTINCT FROM bl2.transactionid
                     AND bl.classid IS NOT DISTINCT FROM bl2.classid
                     AND bl.objid IS NOT DISTINCT FROM bl2.objid
                     AND bl.objsubid IS NOT DISTINCT FROM bl2.objsubid
   JOIN pg_stat_activity blocking ON bl2.pid = blocking.pid
   WHERE NOT bl.granted;
   ```
   `blocked_pid` is waiting; `blocking_pid` holds the lock.
3. **Decide before killing.** Inspect `blocking_query`: if it is an essential transaction or a write in progress, **do not terminate** — wait, or escalate to the owner. Terminate only if it is a stuck/idle, non-critical session.
4. **Terminate the blocker** (destructive action — confirm the `pid`):
   ```sql
   SELECT pg_terminate_backend(<blocking_pid>);
   ```
   Try `pg_cancel_backend(<pid>)` first: it cancels the query without dropping the connection.

## Verification

- Re-run the step 2 query: the blocked-pair row is gone.
- Step 1: the `blocked_pid` is no longer in `wait_event_type = 'Lock'` and has progressed or finished.

## Anti-cases

- The blocker is a planned backup/migration/DDL → do not kill it; let it finish.
- Blocks reappear right after the kill → the root cause is in the application (uncommitted transaction); fix the code, not the symptom.
- Terminating `blocked_pid` instead of `blocking_pid` fixes nothing — confirm which is which.
