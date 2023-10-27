---
layout: default
title: Data manipulation tricks
parent: Data Science
nav_order: 2
---


# Lazy split-reduce-compute-merge strategy

Let's say we have a relatively big parquet file containing 100M rows.

This dataset contains geo-meteorological data. Geolocations are encoded by an h3 index.

h3 indexes are strings hashes refering to given hexagonal areas in the world. These h3 hexagonal cells have a resolution, in other words an order of magnitude edge length.

We can recover latitude/longitude of the center of these cells using the following code:

```python
import h3
# loc_id: str = ...
lat, lon = h3.h3_to_geo(loc_id)
```

In our dataset, we may have a lot of meteorological values for the same location.

This means that one of the most effective strategy to recover latitudes and longitudes is the following:

* Load the data
* Get unique locations
* Compute latitude/longitude
* Merge computed lat/lon back to the dataframe

This can be done the following way in Pandas:

```python
df = pd.read_parquet(path)
df = (
    df
    # Have only one row per loc
    .drop_duplicates()
    # Compute lat/lon on each row (not_vectorized)
    .apply(
        lambda row: (row.loc_id, *h3.h3_to_geo(row.loc_id))
    )
    # Convert the pd.Series of tuples into a DataFrame of 3 columns
    .apply(pd.Series)
    # Name obtained columns
    .rename(columns={0: "loc_id", 1: "lat", 2: "lon"})
    # Get back all data
    .merge(
        df,
        on="loc_id",
        how="right",
    )
)
```

This operation takes 18.5 seconds on my setup.

Polars is particularly efficient at this kind of strategy because of the concept of Lazy evaluation and graph optimization.

```python
import polars as pl
df = pl.scan_parquet(path)
df = (
    df
    .join(
        df
        # Get unique loc_id's
        .select("loc_id")
        .unique()
        # Compute latlon column of type list[str]
        .with_columns(
            latlon=pl.col("loc_id").apply(lambda loc_id: h3.h3_to_geo(loc_id))
        )
        # Extract lat/lon from the computed column
        .with_columns(
            lat=pl.col("latlon").list[0],
            lon=pl.col("latlon").list[1],
        )
        # Remove unnecessary latlon column
        .drop("latlon"),
        # Get bacl original data
        on="loc_id",
        how="left",
    )
)
# Compute
df.collect()
```

This computation takes 4.3 seconds on the same setup.
