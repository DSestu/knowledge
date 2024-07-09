# Misc snippets

# Environment setup

## Create truly clean conda env

```python
conda env create -n ENV_NAME python=3.9 --no-default-packages
```

## Install poetry from an existing path

```python
poetry init    
```

# Misc Python snippets

## Parallel Gzip compression

Needs `mgzip`

```python
import os
import mgzip

with open("original_file.txt", "rb") as original,
    mgzip.open("gzipped_file.txt.gz", "wb", thread=os.cpu_count(), blocksize=2*10**8) as fw:
    fw.write(original.read())
```

# Dask

## Repartition data in memory

Let's say we have a dask dataframe named `ddf`.

After loading, if no specific handling, data will be partitioned in an arbitrary way.

This is not well suited if we often groupby with the same keys, and can lead to OOM of some workers, or other bottlenecks.

This is possible to repartition by a categorical key, but it has to be sorted. The manipulation is as follows:

```python
# In this example, the loc_id refers to a geographical hash, and is a string.
(
    ddf
        # Go into a categorical form
        .astype({"loc_id": "category"})
        # Go to an ordered categorical form
        .assign(loc_id=lambda data: data.loc_id.cat.as_ordered())
        .set_index("loc_id")
        # Repartition the data in order to have 4 locations per partition
        .repartition(npartitions=ddf.loc_id.nunique().compute() // 4)
        # get back to loc_id column
        .reset_index()
        # get back the categorical dtype to an unordered form
        .assign(loc_id=lambda data: data.loc_id.cat.as_unordered())
)
```
