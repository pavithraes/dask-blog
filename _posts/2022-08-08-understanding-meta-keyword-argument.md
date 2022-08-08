---
layout: post
title: Understanding Dask’s meta keyword argument
author: Pavithra Eswaramoorthy
tags: [dataframe]
theme: twitter
---

{% include JB/setup %}

If you have worked with Dask DataFrames or Dask Arrays, you have probably come across the `meta` keyword argument and seen one or more of the following warning/error messages:

```
UserWarning: You did not provide metadata, so Dask is running your function on a small dataset to guess output types. It is possible that Dask will guess incorrectly.
```

```
UserWarning: `meta` is not specified, inferred from partial data. Please provide `meta` if the result is unexpected.
```

```
ValueError: Metadata inference failed in …
```

In this blog post, we will discuss:

- what the is `meta` keyword argument, and
- how to use `meta` effectively

with some minimal examples.

We will look at `meta` mainly in the context of Dask DataFrames, however, similar principles also apply to Dask Arrays.

## What is `meta`?

Before answering this, let's quickly discuss [Dask DataFrames](https://docs.dask.org/en/stable/dataframe.html).

A Dask DataFrame is a lazy object composed of multiple [pandas DataFrames](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.html), where each pandas DataFrame is called a "partition". These are stacked along the index and Dask keeps track of these partitions using “divisions”, which is a tuple representing the start and end index of each partition.

![Dask DataFrame consists of multiple pandas DataFrames](https://docs.dask.org/en/stable/_images/dask-dataframe.svg)

When you create a Dask DataFrame, you usually see something like the following:

```python
>>> import pandas as pd
>>> import dask.dataframe as dd
>>> ddf = dd.DataFrame.from_dict(
...     {
...         "x": range(6),
...         "y": range(10, 16),
...     },
...     npartitions=2,
... )
>>> ddf
Dask DataFrame Structure:
                   x      y
npartitions=2
0              int64  int64
3                ...    ...
5                ...    ...
Dask Name: from_pandas, 2 tasks
```

Here, Dask has created the structure of the DataFrame using some "metadata" information about the _column names_ and their _datatypes_. Dask uses this metadata for understanding Dask operations and creating accurate task graphs (i.e., the logic of your computation).

Internally, this metadata is represented as an empty pandas [DataFrame](https://docs.dask.org/en/stable/dataframe.html) or [Series](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.Series.html), which has the same structure as your output Dask DataFrame. To learn more about how `meta` is defined internally, check out the [DataFrame Internal Design documentation](https://docs.dask.org/en/stable/dataframe-design.html#metadata).

The `meta` keyword argument in various Dask DataFrame functions allows you to explicitly share this metadata information with Dask. Note that the keyword argument is concerned with the metadata of the _output_ of those functions.

To see the actual metadata information for a collection, you can look at the `._meta` attribute:

```python
>>> ddf._meta
Empty DataFrame
Columns: [x, y]
Index: []
```

## How to specify `meta`?

You can specify `meta` in a few different ways, but the recommended way for Dask DataFrame is:

> "An empty `pd.DataFrame` or `pd.Series` that matches the dtypes and column names of the output."
>
> ~ [`DataFrame.apply()` docstring](https://docs.dask.org/en/stable/generated/dask.dataframe.DataFrame.apply.html)

For example:

```python
>>> meta_df = pd.DataFrame(columns=["x", "y"], dtype=int)
>>> meta_df

Empty DataFrame
Columns: [x, y]
Index: []

>>> ddf2 = ddf.apply(lambda x: x, axis=1, meta=meta_df).compute()
>>> ddf2
   x  y
0  0  0
1  1  1
2  2  2
3  0  3
4  1  4
5  2  5
```

The [other ways you can describe `meta`](https://docs.dask.org/en/stable/dataframe-design.html#metadata) are:

- For a DataFrame, you can specify `meta` as a:

  - Python dictionary: `{column_name_1: dtype_1, column_name_2: dtype_2, …}`
  - Iterable of tuples: `[(column_name_1, dtype_1), (columns_name_2, dtype_2, …)]`

  **Note** that while describing `meta` as shown above, using a dictionary or iterable of tuples, the order in which you mention column names is important. Dask will use the same order to create the pandas DataFrame for `meta`. If the orders don't match, you will see the following error:

  ```
  ValueError: The columns in the computed data do not match the columns in the provided metadata
  ```

- For a Series output, you can specify `meta` using a single tuple: `(coulmn_name, dtype)`

You should **not** describe `meta` using just a `dtype` (like: `meta="int64"`), even for scalar outputs. If you do, you will see the following warning:

```
FutureWarning: Meta is not valid, `map_partitions` and `map_overlap` expects output to be a pandas object. Try passing a pandas object as meta or a dict or tuple representing the (name, dtype) of the columns. In the future the meta you passed will not work.
```

During operations like `map_parittions` or `apply` (which uses `map_partitions` internally), Dask coerces the scalar output of each partition into a pandas object. So, the output of functions that take `meta` will never be scalar.

For example:

```python
>>> ddf = ddf.repartition(npartitions=1)
>>> result = ddf.map_partitions(lambda x: len(x)).compute()
>>> type(result)
pandas.core.series.Series
```

Here, the Dask DataFrame `ddf` has only one partition. Hence, `len(x)` on that one partition would result in a scalar output of integer dtype. However, when we compute it, we see a pandas Series. This conforms that Dask is coercing the outputs to pandas objects.

**Another note**, Dask Arrays may not always do this conversion. You can look at the API reference for your particular Array operation for details.

```python
>>> import numpy as np
>>> import dask.array as da
>>> my_arr = da.random.random(10)
>>> my_arr.map_blocks(lambda x: len(x)).compute()
10
```

## `meta` does not _force_ the structure or dtypes

`meta` can be thought of as a suggestion to Dask. Dask uses this `meta` to generate the task graph until it can infer the actual metadata from the values. It **does not** force the output to have the structure or dtype of the specified `meta`.

Consider the following example, and remember that we defined `ddf` with `x` and `y` column names in the previous sections.

If we provide different column names (`a` and `b`) in the `meta` description, Dask uses these new names to create the task graph:

```python
>>> meta_df = pd.DataFrame(columns=["a", "b"], dtype=int)
>>> result = ddf.apply(lambda x:x*x, axis=1, meta=meta_df)
>>> result
Dask DataFrame Structure:
                   a      b
npartitions=2
0              int64  int64
3                ...    ...
5                ...    ...
Dask Name: apply, 4 tasks
```

However, if we compute `result`, we will get the following error:

```
>>> result.compute()
ValueError: The columns in the computed data do not match the columns in the provided metadata
  Extra:   ['x', 'y']
  Missing: ['a', 'b']
```

While computing, Dask evaluates the actual metadata with columns `x` and `y`. This does not match the `meta` that we provided, and hence, Dask raises a helpful error message. Notice how Dask does not change the output to have `a` and `b` here, rather uses `a` and `b` column names only for intermediate task graphs.

## Using `_meta` directly

In some rare case, you can also set the `_meta` attribute directly for a Dask DataFrame. For example, if the DataFrame was read with incorrect dtypes, like:

```python
>>> ddf = dd.DataFrame.from_dict(
...     {
...         "x": range(6),
...         "y": range(10, 16),
...     },
...     dtype="object", # Note the “object” dtype
...     npartitions=2,
... )
>>> ddf
Dask DataFrame Structure:
                    x       y
npartitions=2
0              object  object
3                 ...     ...
5                 ...     ...
Dask Name: from_pandas, 2 tasks
```

The values are clearly integers but the dtype says `object`, so we can't perform integer operations like addition:

```python
>>> add = ddf + 2
ValueError: Metadata inference failed in `add`.

Original error is below:
------------------------
TypeError('can only concatenate str (not "int") to str')
```

Here, we can explicitly define `_meta`:

```python
>>> ddf._meta = pd.DataFrame(columns=["x", "y"], dtype="int64")
```

Then, perform the addition:

```python
>>> add = ddf + 2
>>> add
Dask DataFrame Structure:
                   x      y
npartitions=2
0              int64  int64
3                ...    ...
5                ...    ...
Dask Name: add, 4 tasks
```

Thanks for reading!

Have you run into issues with `meta` before? Please let us know on [Discourse](https://dask.discourse.group/), and we will consider including it here, or updating the Dask documentation. :)
