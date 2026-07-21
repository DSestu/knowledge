
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

## Automatically downcast dtypes of a polars dataframe

```python
def _equal_with_nulls(a: pl.Series, b: pl.Series) -> bool:
    """True if a and b have identical null masks and identical non-null values."""
    if not (a.is_null() == b.is_null()).all():
        return False
    return bool((a.drop_nulls() == b.drop_nulls()).all())

def downcast_df(
    df: pl.DataFrame,
    *,
    downcast_floats: bool = True,
    cast_strings: bool = True,
    cat_max_ratio: float = 0.5,
    downcast_datetimes: bool = True,
) -> pl.DataFrame:
    """Downcast column types to smaller representations without losing data.

    - Integers  -> smallest (u)int type fitting the observed [min, max].
    - Floats    -> Float32 if the round-trip is exact (opt-out via downcast_floats).
    - Strings   -> Categorical when unique ratio <= cat_max_ratio (opt-out via cast_strings).
    - Datetimes -> Date if no time-of-day component; else narrow ns/us -> ms losslessly.
    Null-only and empty columns are left untouched. Other dtypes are ignored.
    """
    int_types = [
        (pl.Int8, -(2**7), 2**7 - 1),
        (pl.UInt8, 0, 2**8 - 1),
        (pl.Int16, -(2**15), 2**15 - 1),
        (pl.UInt16, 0, 2**16 - 1),
        (pl.Int32, -(2**31), 2**31 - 1),
        (pl.UInt32, 0, 2**32 - 1),
        (pl.Int64, -(2**63), 2**63 - 1),
    ]

    out = df
    for name, dtype in df.schema.items():
        col = out.get_column(name)

        if dtype.is_integer():
            lo, hi = col.min(), col.max()
            if lo is None:  # all-null or empty -> nothing to infer
                continue
            for target, tmin, tmax in int_types:
                if lo >= tmin and hi <= tmax:
                    out = out.with_columns(col.cast(target).alias(name))
                    break

        elif dtype == pl.Float64 and downcast_floats:
            if _equal_with_nulls(
                col, col.cast(pl.Float32).cast(pl.Float64)
            ):
                out = out.with_columns(col.cast(pl.Float32).alias(name))

        elif dtype == pl.String and cast_strings:
            n = col.len()
            if n and col.n_unique() / n <= cat_max_ratio:
                out = out.with_columns(
                    col.cast(pl.Categorical).alias(name)
                )

        elif isinstance(dtype, pl.Datetime) and downcast_datetimes:
            if col.drop_nulls().is_empty():
                continue
            # 1) Datetime -> Date when every value sits at midnight (Int64 -> Int32).
            no_time = (
                col.dt.hour().fill_null(0).eq(0)
                & col.dt.minute().fill_null(0).eq(0)
                & col.dt.second().fill_null(0).eq(0)
                & col.dt.nanosecond().fill_null(0).eq(0)
            ).all()
            if no_time:
                out = out.with_columns(col.cast(pl.Date).alias(name))
                continue
            # 2) Otherwise narrow the time unit (ns -> us -> ms) if precision is unused.
            for tu in ("ms", "us"):
                if dtype.time_unit == tu:
                    break  # already at/finer-than a coarser candidate we'd try
                target = pl.Datetime(
                    time_unit=tu, time_zone=dtype.time_zone
                )
                if _equal_with_nulls(col, col.cast(target).cast(dtype)):
                    out = out.with_columns(col.cast(target).alias(name))
                    break

    return out

print(df.estimated_size("mb"), "->", downcast_df(df).estimated_size("mb"))
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

## Sparse design matrix from a tidy polars frame (regression without a dense pivot)

Suppose you have a **long / tidy** table — one row per `(observation, feature)` with
the feature's value — and a single target scalar per observation. You want to regress
the target on the features (e.g. to recover a per-feature coefficient).

The naive move is `df.pivot(...)` into a wide `observation × feature` matrix and feed
that to a regressor. But when there are many features and each observation only touches
a few of them, that matrix is **mostly zeros**: the dense pivot explodes memory
(`n_obs × n_features × 8 bytes`) while carrying almost no information.

Instead, build a `scipy.sparse` matrix **directly** from integer row/column codes
derived in polars — the non-zero triplets only — and hand it to a solver that accepts
sparse input. Nothing dense is ever materialised.

```python
import numpy as np
import polars as pl
import scipy.sparse as sp

# tidy: one row per (observation, feature); `value` is that feature's value for the obs
tidy = pl.DataFrame(
    {
        "obs_id":  ["A", "A", "B", "C", "C", "D", "D", "D", "E", "F", "F"],
        "feature": ["x1", "x3", "x2", "x1", "x2", "x1", "x2", "x3", "x3", "x1", "x2"],
        "value":   [10.0, 4.0, 7.0, 9.0, 3.0, 5.0, 2.0, 1.0, 6.0, 2.0, 8.0],
    }
)
# target: exactly one scalar per observation (here y = x1 + x2 + x3)
targets = pl.DataFrame(
    {"obs_id": ["A", "B", "C", "D", "E", "F"], "y": [14.0, 7.0, 12.0, 8.0, 6.0, 10.0]}
)

# 1) integer row index for observations -> also fixes the target vector order
obs = targets.with_row_index("row")                       # row = 0..n_obs-1
# 2) integer column index for features -> this frame IS the label<->column dictionary
feats = tidy.select("feature").unique().sort("feature").with_row_index("col")

# 3) attach both indices to the tidy frame (inner join drops obs without a target)
long = (
    tidy.join(obs.select("obs_id", "row"), on="obs_id", how="inner")
    .join(feats, on="feature")
)

# 4) build the sparse matrix from (row, col, value) triplets — no dense pivot
X = sp.csr_matrix(
    (long["value"].to_numpy(), (long["row"].to_numpy(), long["col"].to_numpy())),
    shape=(obs.height, feats.height),
)
y = obs["y"].to_numpy()          # aligned to `row` because `obs` defined that order
```

`X` has millions of rows if the data is large, but only the non-zeros are stored.

### Fitting — OLS with inference via the normal equations

The key observation: even if `X` has millions of *rows*, it has only `p` *columns*, so
the normal-equations matrix `XᵀX` is just `p × p` and inverts cheaply. That gives
coefficients **and** standard errors / p-values straight off the sparse matrix, without
a dense design matrix or `statsmodels`:

```python
from scipy import stats

n, p = X.shape
XtX = (X.T @ X).toarray()                 # (p x p) — tiny, dense
Xty = np.asarray(X.T @ y).ravel()
rank = int(np.linalg.matrix_rank(XtX))
XtX_inv = np.linalg.pinv(XtX)             # pinv: robust to collinear / rank-deficient features
beta = XtX_inv @ Xty

resid = y - X @ beta
dof = n - rank
sigma2 = float(resid @ resid) / dof
se = np.sqrt(np.clip(np.diag(XtX_inv) * sigma2, 0, None))
t = np.divide(beta, se, out=np.full_like(beta, np.nan), where=se > 0)
pval = 2.0 * stats.t.sf(np.abs(t), dof)

results = feats.with_columns(
    pl.Series("beta", beta),
    pl.Series("std_err", se),
    pl.Series("p_value", pval),
)
```

### Gotchas

- **Positional alignment.** `beta[j]` is the coefficient of the feature
  whose `col == j`. Because `feats` was built with `with_row_index` in a fixed sorted
  order, dropping `pl.Series(...)` onto it aligns by position — so **never re-sort
  `feats` between building `X` and attaching the coefficients**, and keep `obs`
  immutable so `y` stays aligned to `row`.
- **Collinearity.** Features that never appear apart aren't separately identifiable;
  `pinv` + rank-based `dof` keep the fit stable and those features surface as large
  `std_err` / wide confidence intervals rather than blowing up the inverse.
- **Model choice.** Add `fit_intercept` by appending an all-ones column to `X` if the
  target isn't a pure sum of the features; omit it when it is.
- **Sparse format.** `csr` is fine for `X @ v` and the normal equations; `sklearn`
  linear models also accept sparse input (they prefer `csc` and convert automatically).
