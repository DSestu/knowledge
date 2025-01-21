

# Data manipulation tricks

## Recursively rename columns on a Pandas DataFrame 

```python
df.pipe(
    lambda dataframe: functools.reduce(
        lambda df, string: df.assign(
            window_type=lambda data: data["window_type"].str.replace(string, "")
        ),
        ["_CONDITIONS", "_DEFINITIONS"],
        dataframe,
    )
)
```

## Geographical DBScan, and centroid estimation

```python
import numpy as np
from shapely import MultiPoint
from sklearn.cluster import DBSCAN

locations: pl.DataFrame

# Main variable for DBScan: the kernel size (here roughly in km's)
spatial_kernel = 60 # km

# Estimation of clusters
epsilon = spatial_kernel / kms_per_radian
dbscan = DBSCAN(epsilon, min_samples=1, algorithm="ball_tree", metric="haversine").fit(
    np.radians(locations[["latitude", "longitude"]])
)

# Enrich original dataset with cluster labels
locations = locations.with_columns(
    pl.Series(dbscan.labels_, dtype=pl.String).alias("cluster")
)

# (optional) For each cluster, get the geographical centroid
centroids = []
for (cluster,), _ in unique_locations.group_by("cluster"):
    lat, lon = MultiPoint(
        np.array(_[["latitude_sencrop", "longitude_sencrop"]])
    ).centroid.coords.xy
    lat, lon = [_[0] for _ in [lat, lon]]

    centroids.append({"cluster": cluster, "lat": lat, "lon": lon})
centroids = pl.DataFrame(centroids)
```

## Geographical K-nearest neighbors from lat/lon

Let's say we have a pandas dataframe with latitude (lat) and longitude (lon) columns.

We want for each location the 12 nearest locations of the same dataframe.

First:

```python
from sklearn.neighbors import BallTree
import pandas as pd

from math import radians

EARTH_RADIUS = 6371000 / 1000
```

We need to convert lat/lon to radians:

```python
df = (
    df
    .assign(
        lat_rad = lambda data: data.latitude.apply(radians),
        lon_rad = lambda data: data.longitude.apply(radians),
    )
)
```

BallTree returns 2 arrays:

* One of dim N-locations x N-neighbors, with distances values
* The other also of dim N-locations x N-neighbors, with values values being indexes designating the neighbor

Here, we want 12 neighbors:

```python
distances, neighbors = BallTree(
    df[["lat_rad", "lon_rad"]].to_numpy(),
    metric="haversine"
).query(
    df[["lat_rad", "lon_rad"]].to_numpy(),
    # 13 because it includes itself
    k=13,
)
```

To match indexes with distances, we will melt both arrays before a join.
This will give a dataframe with: index/neighbor_index/distance in meters.

```python
df_distance = (
    pd.DataFrame(distances)
        # 0 is the origin city, drop it
        .drop(columns=[0])
        .reset_index()
        .melt(id_vars="index")
        .rename(
            columns={
                "value": "distance",
            },
        )
        # Get distance in rounded meters
        .assign(
            distance=lambda data: (data["distance"] * EARTH_RADIUS * 1000).astype(int)
        )
        .merge(
            pd.DataFrame(neighbors)
            .drop(columns=[0])
            .reset_index()
            .melt(id_vars="index")
            .rename(
                columns={
                    "value": "neighbor_index",
                }
            ),
            on=["index", "variable"],
            how="left"
        )
)
```

Now, if we want to recover some data from additional columns of our original dataset, we can do this:

```python
df_export = (
    df
        .reset_index()
        .merge(
            df_distance,
            on="index",
        )
        .merge(
            df
            .reset_index()
            .rename(columns={"index": "neighbor_index"}),
            on="neighbor_index",
            suffixes=["", "_neighbor"]
        )
)
```

## Lazy split-reduce-compute-merge strategy

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

This computation takes 1.5 seconds on the same setup.
