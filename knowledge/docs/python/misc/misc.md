# Misc snippets

# ZSHrc snippet to pull every repo in a folder in parallel

Put this in the `.zshrc`.

This supposes that every repo is in `$HOME/github/`, change that accordingly.

```zsh
alias pullall='for d in $HOME/github/*/; do (cd "$d" && if [ -d .git ]; then echo "Pulling in $(pwd)" && git pull; fi) & done; wait'

```

# Reading environment file and modifying env for single command

```bash
env $(cat <.env-file> | grep -v "#" | xargs) <command>
```

What does this do ?

* `env` modifies the environment variables for a single command. It takes args like `ENV_VAR1=a ENV_VAR2=b`

* `$(...)` takes the stdout of the `...` command to inject it as a string.

* `cat` OFC stdout the content of the file

* `grep -v "#"` removes every line that is commented

* `xargs` takes the output of grep an normalize its whitespaces etc so it passes each env row as `ENV_VAR1=a ENV_VAR2=b` etc

# Making Synchronous code like asynchronous with threads

The following code will run for 5 seconds and then print "Finished sleeping" both at the same time

```python
import asyncio
import functools
import time
from concurrent.futures import ThreadPoolExecutor

# Global executor that can be reused
thread_pool = ThreadPoolExecutor()


def make_async(func: callable) -> callable:
    """Decorator to convert any sync function to async function"""

    @functools.wraps(func)
    async def wrapper(*args: tuple, **kwargs: dict) -> any:
        loop = asyncio.get_running_loop()
        return await loop.run_in_executor(thread_pool, lambda: func(*args, **kwargs))

    return wrapper


@make_async
def sleep(n):
    time.sleep(n)
    print("Finished sleeping")


await asyncio.gather(sleep(5), sleep(5))
```

# Auto-reload an app with a file watchdog

```bash
pip install pytest-watch
ptw --runner "my command that will be re-run if a file change file.py"
```

# Profiling functions

Get a flamegraph of functions inside a notebook

```bash
pip install snakeviz
```

In the notebook

```python
%load_ext snakeviz
%snakeviz_config -h localhost -p 8888
%snakeviz myfunctions
```

# LRU caching with custom hashing function

You may be in the situation where you want to have a cache for function calls.

A specific case of those caches is called LRU, which stands for Least Recent Unit.

Least Recent Unit caches have a maximum size. Once the cache is full, the next cached element will involve freeing the least used element in the cache.

This can easily be done with a native python decorator

```python
@lru_cache(maxsize=128)
```

How does this work?

This is simply a save system using a dictionary, with a litle bit of sugar to know which key was the least used.

The dictionnary is used as the following:

* The function is called with arguments

* The output of the function is computed if not in cache

* The arguments and the output are stored in the dictionnary. **The key used to store in the dictionnary is a tuple of the arguments, and the value is the output of the function**

> **The issue here is that the key of a dictionnary need to be hashable!**

**BUT**

Some python objects are not hashable! This means that they can't be cached that way.

There are 2 solutions:

1. If an argument is a custom class

You can define the `.__hash__()` function in your class, and use the standard `@lru_cache`

```python
class MyClass:
    def __init__(self, a: int, b: int) -> None:
        self.a = a
        self.b = b

    def __hash__(self) -> str:
        return str(self.a) + str(self.b)

@lru_cache(maxsize=128)
def get_sum(the_class: MyClass) -> int:
    return the_class.a + the_class.b

my_class = MyClass(a=1, b=2)

for _ in range(1000):
    get_sum(the_class=my_class)

```

2. If you aren't using a custom class, and so can't override the `.__hash__()` function.

This is for example the case of a function that uses operations on a numpy array.

This is a case that I encountered while coding a Connect4 bot. By searching recursively board states, I may encounter multiple times the same board. I don't want to compute scores etc when I already did.

So, I need to cache the result of my function, which has the board as an argument.

But a Numpy array don't have a `.__hash__()`, so I need to provide a custom hashing function that will transform each board in a unique hash.

The hashing function can be whatever, but need to have 2 specific properties:

* The hash has to be deterministic

* One hash can correspond only to one board state

The way I do it is the following:

```python
import numpy as np

def hash_array(arr: np.ndarray) -> int:
    """Used to hash numpy array so we can cache scores inside a dictionary"""
    _arr = arr.copy()
    _arr.flags.writeable = False
    return hash(_arr.tobytes())
```

And I make an object that will be used as a cache:

```python
from collections import OrderedDict

class LRUCacheDict:
    def __init__(self, maxsize=128):
        self.cache = OrderedDict()
        self.maxsize = maxsize

    def get(self, key):
        try:
            value = self.cache.pop(key)
            # Reinsert the key to mark it as recently used
            self.cache[key] = value
            return value
        except KeyError:
            return None

    def put(self, key, value):
        try:
            # Remove the key if it already exists to update its position
            self.cache.pop(key)
        except KeyError:
            if len(self.cache) >= self.maxsize:
                # Pop the first item (the least recently used)
                self.cache.popitem(last=False)
        self.cache[key] = value

    def __repr__(self):
        return repr(self.cache)

    def __len__(self):
        return len(self.cache)

    def __contains__(self, key):
        return key in self.cache

    def clear(self):
        self.cache.clear()
```

And I define the decorator that will use the cache

```python
from functools import wraps

def custom_lru_cache(cache: LRUCacheDict):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            _hash = hash_array(*args)
            value = cache.get(_hash)
            if value is not None:
                return value
            else:
                value = func(*args, **kwargs)
                cache.put(_hash, value)
                return value
        return wrapper
    return decorator
```

Now I can use it the following way:

```python
my_array = np.ones(shape=(3, 3))

@custom_lru_cache(LRUCacheDict(maxsize=128))
def get_sum(arr: np.ndarray) -> float:
    return arr.sum()

for _ in range(1000):
    get_sum(my_array)
```

# Environment setup

## Create truly clean conda env

```python
conda env create -n ENV_NAME python=3.9 --no-default-packages
```

## Adding custom *(local)* packages to a specific conda environment

Works with `conda` or `mamba`.

```bash
mamba env config vars list # Lists actual custom environment-dependent variables
mamba env config vars set PYTHONPATH=/full/path/to/module # Make sure to use the full path, no ~
mamba env config vars unset PYTHONPATH # Removes a variable
```

## Install poetry from an existing path

```python
poetry init    
```

# GIT

## Permanently ignore changes on a file

In some cases we don't want to gitignore a file. This can happen if you work with multiple people, and an IDE extension always creates a file. The file creation is only happenning to you, and you don't want to add too much pollution in the gitignore.

In this case, you can permanently ignore the change of a file with the following snippet

```bash
git update-index --assume-unchanged FILE
```

## Pushing to both Github and Huggingface

```bash
git remote add origin git@(...).git
git remote set-url --push origin git@(...).git
git remote set-url --add --push origin https://{user}:{token}@huggingface.co/spaces/{user}/{space}
git remote -v
git push --set-upstream origin main
```

## Purge and clean local .git folder

```bash
git gc --prune=now
git repack -Ad
git prune
```

```bash
git gc --prune=now && git repack -Ad && git prune
```

## Check sanity of local repo

```bash
git gc --prune=now
```

## Removing untracked files

for dry run use `-n` parameter

### Remove EVERY untracked file (including gitignored)

```bash
git clean -fdx
```

### Remove untracked file (but keep gitignored)

```bash
git clean -fd
```

### Remove untracked files (only gitignored ones)

```bash
git clean -fdX
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
