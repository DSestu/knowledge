# Snippets

## Show running queries & locks blocking them with the nature and source of the lock

```sql
SELECT
    a.pid,
    a.usename AS user,
    a.application_name,
    a.client_addr,
    a.client_port,
    a.state,
    a.query,
    a.query_start AS query_start_time,
    age(now(), a.query_start) AS duration,
    a.wait_event_type,
    a.wait_event,
    (
        SELECT string_agg(
            l2.locktype || ':' || 
            COALESCE((l2.relation::regclass)::text, '') || 
            COALESCE(':tuple=' || l2.tuple::text, ''),
            ', '
        )
        FROM pg_locks l2
        WHERE l2.pid = a.pid AND l2.granted = false
    ) AS locking_on,
    (
        SELECT string_agg(
            blocker.pid::text, ', '
        )
        FROM pg_locks blocked
        JOIN pg_locks blocker ON blocker.locktype = blocked.locktype
            AND blocker.DATABASE IS NOT DISTINCT FROM blocked.DATABASE
            AND blocker.relation IS NOT DISTINCT FROM blocked.relation
            AND blocker.page IS NOT DISTINCT FROM blocked.page
            AND blocker.tuple IS NOT DISTINCT FROM blocked.tuple
            AND blocker.virtualxid IS NOT DISTINCT FROM blocked.virtualxid
            AND blocker.transactionid IS NOT DISTINCT FROM blocked.transactionid
            AND blocker.classid IS NOT DISTINCT FROM blocked.classid
            AND blocker.objid IS NOT DISTINCT FROM blocked.objid
            AND blocker.objsubid IS NOT DISTINCT FROM blocked.objsubid
        WHERE blocked.pid = a.pid AND NOT blocked.granted AND blocker.granted
    ) AS blocked_by
FROM
    pg_stat_activity a
WHERE
    a.state <> 'idle'
    AND a.query NOT ILIKE '%pg_stat_activity%' -- Exclude this monitoring query itself
ORDER BY
    duration DESC;
```

## List manually created tables

```sql
SELECT 
  nsp.nspname as object_schema,
  cls.relname as object_name, 
  rol.rolname as owner, 
  CASE cls.relkind
    WHEN 'r' then 'TABLE'
    WHEN 'm' then 'MATERIALIZED_VIEW'
    WHEN 'i' then 'INDEX'
    WHEN 'S' then 'SEQUENCE'
    WHEN 'v' then 'VIEW'
    WHEN 'c' then 'TYPE'
    ELSE cls.relkind::text
    END AS object_type
FROM 
  pg_class cls
  JOIN pg_roles rol ON rol.oid = cls.relowner
  JOIN pg_namespace nsp ON nsp.oid = cls.relnamespace
WHERE nsp.nspname NOT IN ('information_schema', 'pg_catalog')
  AND nsp.nspname NOT LIKE 'pg_toast%'
  AND rol.rolname NOT IN  ('processor', 'rdsadmin', 'rds_superuser', 'postgres')
ORDER BY 
  nsp.nspname, 
  cls.relname
```

## Credits to [rgreenjr](https://gist.github.com/rgreenjr/3637525)

### Show running queries (9.2)

```sql
SELECT 
    pid, 
    age(clock_timestamp(), query_start), 
    usename, 
    query 
FROM 
    pg_stat_activity 
WHERE 
    query != '<IDLE>' 
    AND query NOT ILIKE '%pg_stat_activity%' 
ORDER BY 
    query_start desc
```

### Kill running query

```sql
SELECT pg_cancel_backend(procpid);
```

### Kill idle query

```sql
SELECT pg_terminate_backend(procpid);
```

### Vacuum command

```sql
VACUUM (VERBOSE, ANALYZE);
```

### All database users

```sql
SELECT 
    * 
FROM 
    pg_stat_activity 
WHERE 
    current_query NOT LIKE '<%'
```

### All databases and their sizes

```sql
SELECT * FROM pg_user;
```

### All tables and their size, with/without indexes

```sql
SELECT 
    datname, 
    pg_size_pretty(pg_database_size(datname))
FROM 
    pg_database
ORDER BY 
    pg_database_size(datname) DESC
```

### Cache hit rates (should not be less than 0.99)

```sql
SELECT 
    SUM(heap_blks_read) AS heap_read, 
    SUM(heap_blks_hit)  AS heap_hit, 
    (sum(heap_blks_hit) - sum(heap_blks_read)) / sum(heap_blks_hit) AS ratio
FROM 
    pg_statio_user_tables
```

### Table index usage rates (should not be less than 0.99)

```sql
SELECT 
    relname, 
    100 * idx_scan / (seq_scan + idx_scan) AS percent_of_times_index_used, 
    n_live_tup AS rows_in_table
FROM 
    pg_stat_user_tables 
ORDER BY 
    n_live_tup DESC
```

### How many indexes are in cache

```sql
SELECT 
    SUM(idx_blks_read) AS idx_read, 
    SUM(idx_blks_hit)  AS idx_hit, 
    (SUM(idx_blks_hit) - SUM(idx_blks_read)) / SUM(idx_blks_hit) AS ratio
FROM 
    pg_statio_user_indexes
```
