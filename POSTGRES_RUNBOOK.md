# Postgres Stats Runbook

## Ranking by User (Top Consumers)

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

**What to look for**

* `total_exec_time_ms` → which user account is consuming the most database time overall.
* `avg_time_per_call_ms` → which user’s queries are individually slow.
* This helps identify specific services or applications causing heavy load.

## Top Queries by Total Execution Time

```sql
SELECT query,
       calls,
       total_exec_time AS total_ms,
       mean_exec_time AS avg_ms,
       rows,
       round((total_exec_time / calls)::numeric, 2) AS avg_time_per_call_ms
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

**What to look for**

* High `total_exec_time` → query consuming the most total time (likely repeated and expensive).
* High `avg_time_per_call_ms` → query that’s individually slow per execution.
* Combine with `calls` to identify if it’s slow due to frequency or complexity.

## Active Sessions

Check which queries are currently running:

```sql
SELECT pid,
       usename,
       datname,
       state,
       wait_event_type,
       wait_event,
       query_start,
       now() - query_start AS execution_time,
       query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY execution_time DESC;
```

**What to look for**

* `execution_time` → how long the query has been running.
* `wait_event_type = 'Lock'` → indicates that the query is waiting on a lock (possible blocking).
* `state = 'idle in transaction'` → may indicate an open transaction left hanging (potential source of locks).

## Identify Blocked and Blocking Queries

```sql
SELECT blocked.pid     AS blocked_pid,
       blocked.usename AS blocked_user,
       blocked.query   AS blocked_query,
       blocking.pid    AS blocking_pid,
       blocking.usename AS blocking_user,
       blocking.query   AS blocking_query
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

**Result:**
Shows which queries are blocked and which ones are causing the block.

If you need to terminate a blocked session:

```sql
SELECT pg_terminate_backend(<pid>);
```

**What to look for**

* `blocked_pid` → the process waiting on a lock.
* `blocking_pid` → the process holding the lock.
* Terminate the blocking session **only if** it’s not critical or part of an essential transaction.

## Top Queries by Disk I/O

```sql
SELECT query,
       shared_blks_read,
       shared_blks_hit,
       round(100 * shared_blks_hit::numeric / nullif(shared_blks_hit + shared_blks_read, 0), 2) AS hit_ratio
FROM pg_stat_statements
ORDER BY shared_blks_read DESC
LIMIT 10;
```

**What to look for**

* `shared_blks_read` → high value means many reads from disk (possible missing index).
* `hit_ratio` → lower values (<90%) indicate inefficient caching — the query often reads from disk.

## Reset Statistics (for Fresh Analysis)

```sql
SELECT pg_stat_statements_reset();
```

Do this before a controlled testing window to monitor only new queries.

**What to look for**

* Use after deployments or performance tests to get clean metrics.

