# Pyspark snippets

# Misc

## Local Pytest

When using pytest routines with PySpark, you may encounter a situation where the pyspark interpreter don't properly recognize python installation.

This can be fixed by integrating (*temporarly*) a environment variable in the `__init__.py` located at the root of your testing folder:

```python
import os
os.environ[
    "PYSPARK_PYTHON"
] = r"C:\micromamba\.micromamba\envs\weather-pipelines\python.exe"
```

# SQL

## Getting data

using a sql like query.

```python
spark.sql(query)
```

## Some syntaxes differs from postgresql

> Check the elements of an array:

```sql
array_contains(flags_array, 'REFILL')
```

> Check the size of an array

```sql
size(flags_array) = 0
```

# Scalars & arrays

## Arrays

### Making a "set"/array

```python
array()
```

### Sorting an array

> Sorts by alphabetical order. Useful for "hashing".

```python
sort_array(array("col1", "col2"...))
```

### Getting the size of an array

```python
size(array)
```

### Intersection between multiple arrays

```python
array_intersect(array1, array2)
```

## Scalars

> The big difference between PySpark and conventionnal Python is the handling of scalars:

- *In Python*: you can just multiply an array by a value, or pass whatever value to a function.

- *In PySpark*: you have to use `lit()` in order to pass arguments, or multiply by a value. It will enable distribution of the value across workers.

```python
# python
array * 3
# pyspark
array * lit(3) 
```

# Some equivalences with pandas syntax

## The `col()` cursor

> To make a reference to a column, you have to use `col()`, which acts kind like a cursor.

During method chaining, the `col()` is aware of the *`self`* which is passed to the method.

Example:

```python
df.filter(col("column_name") == "abc")
```

You can also reference directly the column in a pandas style, but still have to reference it inside a `col()`:

```python
df.filter(col(df.column_name) == "abc")
```

You can also reference columns from another dataframe, note that using a **join strategy would be greatly better**:

```python
df.filter(col(another_df.column_name) == "abc")
```

## Checking null values

Creating a new boolean column based on the nullity of another.

```python
df.withColumn(
    "new_column", col("old_column").isNull()
)
```

## Merging

> **join** is the equivalent of **merge** in pandas.

Here: joining 2 dataframes and then filtering based on condition.

Case when there are columns with the same name in left and right.

> **You can also join on conditions**. You have to use conditions if the naming of the columns is not the same between each dataframe.

```python
df.join(
    df2,
    on="loc_id",
    how="left",
).where(
    df2["loc_id"].isNull(),
    # Equivalent au WHERE sql
)
```

## Filtering data

2 alternatives: These function operates in an identical way.

|command  |pandas equivalent  |
|---------|---------|
|df.where(col("col1") == 0) *or* df.where("col1" == 0) *or* df.where(col(df.col1) == 0)  |df.loc[df.col1 == 0]         |
|df.where("col1" == 0)|df.loc[df.col1 == 0]         |
|df.where(col(df.col1) == 0)  |df.loc[df.col1 == 0]         |
|df.filter("col1 == 0")     |df.query("col1 == 0")         |
|df.filter(conditions)     |df.query("col1 == 0")         |

> Conditions have the same syntax than in pandas / conventional Python.

|operator  |meaning  |
|---------|---------|
|`&`     |and         |
|`\|`     |or         |
|`~`     |not         |

Example:

```python
condition = (
    (
        # Col 1 equals 0 and col 2 equals 1
        col("col1") == 0 &
        col("col2") == 1    
    ) | # OR
    ~(
        # Not col 1 equals 1 and col 2 equals 0
        col("col1") == 1 &
        col("col2") == 0
    )
)
df = df.filter(condition)
```

## `select()` is equivalent to `get()` in pandas

```python
# Pandas
df.get(["col1", "col2", "col3",])
# PySpark
df.select("col1", "col2", "col3")
df.select(col("col1"), col("col2"), col("col3"))

```

## Dropping a column

Almost the same syntax:

```python
# Pandas
df.drop(columns=["col1", "col2", "col3"])
# PySpark
df.drop("col1", "col2", "col3")
```

## Alias

By default, every operation on a column will modify its name:

Example a `sum` of `value` will have the name `sum(value)`.

In order to rename a column, you have to select it with an alias.

```python
df.select(col("column").alias("new_column_name"))
```

## Explode

> Same syntax as in pandas

Note: you can explode directly in `select`.

```python
# Pandas
df.groupby(["col1", "col2", "col3"]).explode("col3")
# PySpark
df.select("col1", "col2", explode("col3").alias("new_col3_name"))
```

## Getting column names

Exactly the same as pandas:

```python
df.columns
```

## Drop duplicates

Exactly the same as pandas (in camelcase)

```python
df.dropDuplicates(subset=["col1", "col2", ...])
```

## Creating a column in an assign style (withColumn)

Downside of `withColumn` is that it is used to create only one column at a time.

```python
# Python
df.assign(
    new_column=lambda data: data.old_numeric_column.sum()
)
# PySpark
df.withColumn(
    "new_column",
    sum(col("old_numeric_column"))
)
```

## GroupBy Agg

Very similar syntax to pandas:

```python
# Python
(
    df
        .groupby("col1", as_index=False)
        .agg(nb_of_days=("day", "nunique"))
)
# PySpark
(
    df
        .groupBy("col1")
        .agg(
            collect_set( 
                # Equivalent of "nunique" method
                "day",
            ).alias(
                # Name the output column
                "nb_of_days"
            )
        )
)
```

## expr

> `expr()` is kind of like `eval()` in pandas

```python
df.select(
    df.date,
    df.increment,
    expr("add_months(date,increment)").alias("incremented_date")
)
```

## Melt

```python
# Pandas
df.get(["col1", "col2", "col3"]).melt(id_vars="col1")
# Pyspark
df.melt(
    ids="col1"
    values=["col2", "col3"]
    variableColumnName="variable",
    valueColumnName="value",
)
```

## Cast as timestamp

```python
# Python
pd.to_datetime(df.timestamp)
# PySpark
to_timestamp(col("timestamp"))
```

## Timestamp to date

```python
# Pandas
df.timestamp.dt.floor("D")
# Pyspark
date_format(col(df.timestamp), "D")
```

## Date to day of the year

```python
# Pandas 
df.timestamp.dt.day_of_year
# PySpark
col("timestamp").cast("integer")
```

## Create bins

```python
bucketize = Bucketizer(
    splits=value_bins,
    inputCol="value",
    outputCol="category_value",
)
df = bucketize.transform(df)
```

## Gathering a pandas dataframe

```python
df.toPandas()
```

> This will provoke a compute !!

## `.show()` equivalent to `.head()` in pandas

> `.show()` is equivalent to `.head(20)` in Pandas.

It computes a part of the result.

## H3

### Geo to h3

```python
h3_pyspark.geo_to_h3("lat", "lon", lit(grid_resolution))
```

### Distance between locations

```python
h3_pyspark.point_dist(loc1, loc2, lit("m"))
```

## Saving data in the metastore

```python
database = "hive_metastore.weather_analyses"
filename = "..."

df.write.saveAsTable(
    name=f"{database}.{filename}",
    mode="overwrite"
)
```

> Another (more generalist) syntax may exist une autre syntaxe for the `overwrite`.
