Title: Pandas Extension Arrays
Date: 2019-01-04

Extensibility has been of the major themes in pandas development over the last
couple of releases. This post introduces the extension array interface: the
motivation behind them and how they might affect you as a pandas user. Finally,
we look at how extension arrays shape the future of pandas.

## The Motivation

Pandas is built on top of NumPy. You could roughly define a Series as a wrapper
around a NumPy array, and a DataFrame as a collection of Series with a shared
index. That's not entirely correct for several reasons, but I want to focus on
the "wrapper around a NumPy array" part. It'd be more correct to say "wrapper
around an array-like object".

While pandas mostly used NumPy's type system, we restricted it in places and
extended it in others. For example, pandas' early users cared greatly about
timezone-aware datetimes, which NumPy doesn't support. So pandas internally
defined a `DatetimeTZ` dtype (which mimics a NumPy dtype), and allowed you to
use that dtype in `Index`, `Series`, and as a column in a `DataFrame`. That
dtype carried around the tzinfo, but wasn't itself a valid NumPy dtype.

As another example, consider `Categorical`. This is a class that actually
composes *two* arrays: one for the `categories` and one for the `codes`. And has
a special ``.dtype`` that carries the ``categories`` and ``ordered`` data.

Each of these extension types pandas added is useful on its own, but carries a
high maintenance cost. Large sections of the code base need to be aware of how
to handle a NumPy array, or one of these other kinds of special arrays. This
made adding new extension types to pandas a very high hurdle.

Anaconda, Inc. had a client who regularly dealt with datasets that included IP
addresses. I proposed an [IPArray][IPArray], but the community was hesitant to
take on the maintainence burden of adding another custom dtype. So rather than
accept an `IPArray` in pandas itself, we decided to implement an interface. Any
object implementing this interface would be allowed in pandas. We'd let it into
Series or DataFrame without coercing it, and would dispatch to it whenever
relevant. I was able to write [cyberpandas][cyberpandas] *outside* of pandas,
but it still feels like your normal pandas experience.

## The Current State

As of pandas 0.24.0, all of pandas' internal extension arrays (Categorical,
Datetime with Timezone, Period, Interval, and Sparse) are built on top of the
ExtensionArray interface. Users shouldn't notice many changes (other than
unfortunate API breaking changes like `Series.unique` returning a
`DatetimeArray` for timezone-aware data). The main thing you'll notice is that
things are cast to `object` dtype in fewer places, meaning your code will run
faster and your types will be more stable. This includes storing `Period` and
`Interval` data in `Series` (which were previously cast to object).

One thing it does slightly change is what we'd consider "idiomatic" pandas code.
Occasionally, you'll need to do an operation on a raw (unlabeled) array, rather
than on a Series or DataFrame. Perhaps the method you're calling only works with
NumPy arrays, or perhaps you want to disable automatic alignment.

In the past, you'd hear things like "Use `.values` to extract the NumPy array
from a Series or DataFrame." If it were a good resource, they'd tell you that's
not *entirely* true, since there are some exceptions. I'd like to delve into
those exceptions now.

`.values` overloaded two purposes:

1. Extracting the array backing a Series, Index, or DataFrame
2. Converting the Series, Index, or DataFrame to a NumPy array

As we saw above, the "array" backing a Series or Index might not be a NumPy
array, it may an extension array (from pandas or a third-party library). For
`Categorical`,

```python
>>> cat = pd.Categorical(['a', 'b', 'a'], categories=['a', 'b', 'c'])
>>> ser = pd.Series(cat)
>>> ser
0    a
1    b
2    a
dtype: category
Categories (3, object): [a, b, c]

>>> ser.values
[a, b, a]
Categories (3, object): [a, b, c]
>>> ser = pd.Series(
```

So, `.values` isn't always a NumPy array. Even for dtypes that do return a NumPy
array, it may not be what you want. For period-dtype data, `.values` returns a
NumPy array of `Period` objects, which is expensive to create. For
timezone-aware data, `.values` converts to UTC and *drops* the timezone info.
These kind of surprises stem from trying to shoehorn these extension arrays into
a NumPy array, when the entire point of an extension array is for representing
data NumPy can't natively represent.

We've split the two roles previously occupied by `.values` into two separate
methods.

1. Use `.array` to get a zero-copy reference to the underlying data
2. Use `.to_numpy()` to get a (potentially expensive, lossy) NumPy array of the
   data.
  
So with our Categorical example,

```python
>>> ser.array
[a, b, a]
Categories (3, object): [a, b, c]

>>> ser.to_numpy()
array(['a', 'b', 'a'], dtype=object)
```
  
`.array` will *always* be a an ExtensionArray, and is always a zero-copy
reference back to the data.

`.to_numpy()` is *always* a NumPy array, so you can reliably call
ndarray-specific methods on it.
 

To summarize,

- Use `.to_numpy()` if you absolutely need a NumPy array, and are willing to
  copy or coerce data, or lose type information in the process
- Use `.array` if you need a zero-copy reference to the underlying data.

## Possible Future Paths

I'd like to emphasize that this is an *interface*, and not a concrete array
implementation. We are *not* reimplementing NumPy here in pandas. Rather, this
is a way to take any array-like data structure (one or more NumPy arrays, an
Apache Arrow array, a CuPy array) and place it inside a DataFrame. I think
getting pandas out of the array business, and instead thinking about
higher-level tabular data things, is a health development for the project.

Extension arrays provide some interesting opportunities for pandas. 0.24.0
introduced `IntegerArray`, a nullable-integer dtype, solving one of
pandas-longest running feature requests.


[IPArray]: https://github.com/pandas-dev/pandas/issues/18767
[cyberpandas]: https://cyberpandas.readthedocs.io
