# Kedro 

# Debugging kedro pipelines

## Access to kedro from a notebook

One easy way to debug kedro pipelines is via Notebook.

You can launch a notebook in your environment by using

```bash
kedro jupyter
```

or, if like me you work in VSCode, you can do the following:

from the kedro project root directory

```bash
jupyter server
```

In VSCode, use a notebook, and `Connect to an existing jupyter server` with the address `http://localhost:8888`.

You will find the kedro kernel running in the dropdown list of available kernels.

If you didn't set your jupyter server password before, the URL may have a token, which consumes times to copy/paste. You can also define a permanent password for your jupyter server so you don't have to change the address of the server every time with `jupyter server password`.

### Running a pipeline

The freshly instantiated jupyter kernels has some pre-existing variables (see also: <https://docs.kedro.org/en/stable/notebooks_and_ipython/kedro_and_notebooks.html>):

- catalog
- context
- pipelines
- session

The pipelines variables contains ones that were declared in the pipeline registry.

In order to run a pipeline you can use:

```python
session.run(pipeline_name)
```

However, if ran multiple times, you will encounter this error: `KedroSessionError: A run has already been completed as part of the active KedroSession`.

In order to re-run a same pipeline multiple times, you need to run it on different sessions:

```python
from kedro.framework.session import KedroSession

with KedroSession.create(project_path=my_project_path) as session:
    session.run(pipeline_name)
```

### Running a chunk of pipeline, gathering the output as Python objects

You can have a lot of different nodes in one pipeline.

This becomes very boring and time-consuming to write every node output as an intermediary dataset in the data catalog.

In order to avoid that, you can use the following solution:

In the notebook, you can run a pipeline to get a specific node output:

```python
with KedroSession.create(project_path=project_path) as session:
    output = session.run(pipeline_name, to_outputs=["min_timestamp", "max_timestamp"])
```

The output will be a dictionnary `output_name: output_object`.

This works aswell for spark dataframes, meaning that you can directly use:

```python
output["spark_dataframe"].show()
```

> Please note that if you want the notebook to be in sync with your codebase, you will have to use the `%reload_kedro` magic!

> This debugging process is highly effective as there is no need to restart the notebook kernel in order to have an up-to-date codebase. This means that you can use a notebook in parallel of your development in order to check the impact of the modifications of your code to the actual data.

# Working Spark 3.4/kedro environment

> **Warning:** If you are on Windows, using WSL2 as the micromamba/conda source, make sure to **DISABLE** any antivirus during the install process (avoiding *kernel heap memory corruption*). And **re-enable** it after (so the controlled access folder policy works again, so you will be able to access to files in jupyter).

```bash
micromamba env create -f environmnent.yml
```

`environment.yml`

```yaml
name: wp
channels:
  - conda-forge
  - bioconda
  - defaults
dependencies:
  - pip
  - python==3.10.*
  - pyspark==3.4.1
  - h3-pyspark>=1.2.0
  - shapely>=2.0.*
  - kedro~=0.18.0
  - java-jdk>=8.0.0
  - black
  - flake8
  - jupyter
  - mypy
  - pip:
      - kedro-datasets[spark,pandas]==1.8.*
      - pytest
      - pytest-cov
      - pytest-mock
      - databricks-cli
      - kedro-viz
      - hypothesis
```
