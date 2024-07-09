# Snippets 

## List manually created tables

```
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

```
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

```
SELECT pg_cancel_backend(procpid);
```

### Kill idle query

```
SELECT pg_terminate_backend(procpid);
```

### Vacuum command

```
VACUUM (VERBOSE, ANALYZE);
```

### All database users

```
SELECT 
    * 
FROM 
    pg_stat_activity 
WHERE 
    current_query NOT LIKE '<%'
```

### All databases and their sizes

```
SELECT * FROM pg_user;
```

### All tables and their size, with/without indexes

```
SELECT 
    datname, 
    pg_size_pretty(pg_database_size(datname))
FROM 
    pg_database
ORDER BY 
    pg_database_size(datname) DESC
```

### Cache hit rates (should not be less than 0.99)

```
SELECT 
    SUM(heap_blks_read) AS heap_read, 
    SUM(heap_blks_hit)  AS heap_hit, 
    (sum(heap_blks_hit) - sum(heap_blks_read)) / sum(heap_blks_hit) AS ratio
FROM 
    pg_statio_user_tables
```

### Table index usage rates (should not be less than 0.99)

```
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

```
SELECT 
    SUM(idx_blks_read) AS idx_read, 
    SUM(idx_blks_hit)  AS idx_hit, 
    (SUM(idx_blks_hit) - SUM(idx_blks_read)) / SUM(idx_blks_hit) AS ratio
FROM 
    pg_statio_user_indexes
```
