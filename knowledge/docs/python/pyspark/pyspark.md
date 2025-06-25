# Pyspark snippets

# Misc

## Extracting a single batch from a streaming DataFrame as a DataFrame

Useful for debugging purposes.

```python
from typing import Optional
from pyspark.sql import DataFrame, SparkSession
from pyspark.sql.streaming.query import StreamingQuery


def capture_single_microbatch(
    spark: SparkSession,
    stream_df: DataFrame,
) -> Optional[DataFrame]:
    """Capture a single microbatch and return it as a standard DataFrame.

    Args:
        spark: The active Spark session
        stream_df: The streaming DataFrame to capture from

    Returns:
        Optional[DataFrame]: The captured microbatch as a standard DataFrame, 
                           or None if no data was available
    """

    captured_batch = None

    def capture_batch(micro_batch_df: DataFrame, batch_id: int) -> None:
        nonlocal captured_batch
        if micro_batch_df.count() > 0:
            # Cache the DataFrame to avoid recomputation
            captured_batch = micro_batch_df.cache()

    # Process exactly one microbatch
    query = (
        stream_df.writeStream
        .trigger(once=True)
        .foreachBatch(capture_batch)
        .outputMode("append")
        .start()
    )

    # Wait for completion
    query.awaitTermination()

    return captured_batch

my_df = capture_single_microbatch(
    spark=spark, 
    stream_df=stream_df,
)
```

## Local Pytest

When using pytest routines with PySpark, you may encounter a situation where the pyspark interpreter don't properly recognize python installation.

This can be fixed by integrating (*temporarly*) a environment variable in the `__init__.py` located at the root of your testing folder:

```python
import os
os.environ[
    "PYSPARK_PYTHON"
] = r"C:\micromamba\.micromamba\envs\weather-pipelines\python.exe"
```
