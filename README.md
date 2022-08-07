# FilterBox
 

```
pip install filterbox
```

[![tests Actions Status](https://github.com/manimino/filterbox/workflows/tests/badge.svg)](https://github.com/manimino/filterbox/actions)
[![Coverage - 100%](https://img.shields.io/static/v1?label=Coverage&message=100%&color=2ea44f)](test/cov.txt)
[![license - MIT](https://img.shields.io/static/v1?label=license&message=MIT&color=2ea44f)](/LICENSE)
![python - 3.7+](https://img.shields.io/static/v1?label=python&message=3.7%2B&color=2ea44f)


### Usage:

Find which day will be good for flying a kite. It needs to be sunny and windy.

```
from filterbox import FilterBox

days = [
    {'day': 'Saturday', 'sky': 'sunny', 'wind_speed': 1},
    {'day': 'Sunday', 'sky': 'rainy', 'wind_speed': 3},
    {'day': 'Monday', 'sky': 'sunny', 'wind_speed': 7},
    {'day': 'Tuesday', 'sky': 'rainy', 'wind_speed': 9}
]

# define a filter function
def is_windy(obj):
    return obj['wind_speed'] > 5

# make a FilterBox
fb = FilterBox(               # make a FilterBox
    days,                     # provide objects of any Python type
    on=[is_windy, 'sky']      # provide functions + attributes
)

fb.find({is_windy: True, 'sky': 'sunny'})
# result: [{'day': 'Monday', 'sky': 'sunny', 'wind_speed': 7}]
```

Python provides several ways to do this kind of search, such as list comprehension, `filter`, Pandas, and SQLite.
FilterBox is faster than any of these for many kinds of tasks. It can handle as many objects as you have memory for;
typically up to 100 million ~ 1 billion. Find speed stays fast no matter how many objects are in the FilterBox.

There are two classes available.
 - FilterBox: can `add()` and `remove()` objects after creation.
 - FrozenFilterBox: faster finds, lower memory usage, and immutable.

## Examples

Expand for sample code.

<details>
<summary>Match and exclude multiple values</summary>
<br>

```
from filterbox import FilterBox

objects = [
    {'item': 1, 'size': 10, 'flavor': 'melon'}, 
    {'item': 2, 'size': 10, 'flavor': 'lychee'}, 
    {'item': 3, 'size': 20, 'flavor': 'peach'},
    {'item': 4, 'size': 30, 'flavor': 'apple'}
]

fb = FilterBox(objects, on=['size', 'flavor'])

fb.find(
    match={'size': [10, 20]},                # match anything with size in [10, 20] 
    exclude={'flavor': ['lychee', 'peach']}  # where flavor is not in ['lychee', 'peach']
)  
# result: [{'item': 1, 'size': 10, 'flavor': 'melon'}]
```
</details>

<details>
<summary>Accessing nested data using functions</summary>
<br />
Functions are used to get values from nested data structures.

```
from filterbox import FilterBox

objs = [
    {'a': {'b': [1, 2, 3]}},
    {'a': {'b': [4, 5, 6]}}
]

def get_nested(obj):
    return obj['a']['b'][0]

fb = FilterBox(objs, [get_nested])
fb.find({get_nested: 4})  
# result: {'a': {'b': [4, 5, 6]}}
```
</details>

<details>
<summary>Greater than, less than</summary>
<br />

Suppose you need to find objects where x >= some number. If the number is constant, a filter function that returns 
<code>obj.x >= constant</code> will work. 

Otherwise, FilterBox and FrozenFilterBox have a function <code>get_values(attr)</code> which gets the set of 
unique values for an attribute. 

Here's how to use it to find objects having <code>x >= 3</code>.
```
from filterbox import FilterBox

data = [{'x': i} for i in [1, 1, 2, 3, 5]]
fb = FilterBox(data, ['x'])
vals = fb.get_values('x')                # get the set of unique values: {1, 2, 3, 5}
big_vals = [x for x in vals if x >= 3]   # big_vals is [3, 5]
fb.find({'x': big_vals})                 # result: [{'x': 3}, {'x': 5}
```

If x is a float or has many unique values, consider making a function attribute on x that rounds it or puts it
into a bin of similar values. Discretizing x will make lookups faster.
</details>

<details>
<summary>Handling missing attributes</summary>
<br />

Objects don't need to have every attribute.

 - Objects that are missing an attribute will not be stored under that attribute. This saves lots of memory.
 - To find all objects that have an attribute, match the special value <code>ANY</code>. 
 - To find objects missing the attribute, exclude <code>ANY</code>.
 - In functions, raise MissingAttribute to tell FilterBox the object is missing.

Example:
```
from filterbox import FilterBox, ANY
from filterbox.exceptions import MissingAttribute

def get_a(obj):
    try:
        return obj['a']
    except KeyError:
        raise MissingAttribute  # tell FilterBox this attribute is missing

objs = [{'a': 1}, {'a': 2}, {}]
fb = FilterBox(objs, ['a', get_a])

fb.find({'a': ANY})          # result: [{'a': 1}, {'a': 2}]
fb.find({get_a: ANY})        # result: [{'a': 1}, {'a': 2}]
fb.find(exclude={'a': ANY})  # result: [{}]
```
</details>

### Recipes
 
 - [Auto-updating](https://github.com/manimino/filterbox/blob/main/examples/update.py) - Keep FilterBox updated when attribute values change
 - [Wordle solver](https://github.com/manimino/filterbox/blob/main/examples/wordle.ipynb) - Demonstrates using `functools.partials` to make attribute functions
 - [Collision detection](https://github.com/manimino/filterbox/blob/main/examples/collision.py) - Find objects based on type and proximity (grid-based)
 - [Percentiles](https://github.com/manimino/filterbox/blob/main/examples/percentile.py) - Find by percentile (median, p99, etc.)

### API documentation:
 - [FilterBox](https://filterbox.readthedocs.io/en/latest/filterbox.mutable.html#filterbox.mutable.main.FilterBox)
 - [FrozenFilterBox](https://filterbox.readthedocs.io/en/latest/filterbox.frozen.html#filterbox.frozen.main.FrozenFilterBox)

____

## How it works

For every attribute in FilterBox, it holds a dict that maps each unique value to the set of objects with that value. 

FilterBox works something like this: 
```
FilterBox = {
    'attribute1': {val1: {objs}, val2: {more_objs}},
    'attribute2': {val3: {objs}, val4: {more_objs}}
}
```

During `find()`, the object sets matching the query values are retrieved, and set operations like `union`, 
`intersect`, and `difference` are applied to get the final result.

That's a simplified version; for way more detail, See the "how it 
works" pages for [FilterBox](filterbox/mutable/how_it_works.md) and [FrozenFilterBox](filterbox/frozen/how_it_works.md).

### Performance

[Notebook comparing speeds against filter, sqlite, etc.](https://github.com/manimino/filterbox/blob/main/examples/perf_demo.ipynb)

### Related projects

FilterBox is a type of inverted index. It is optimized for its goal of finding in-memory Python objects.

Other Python inverted index implementations are aimed at things like [vector search](https://pypi.org/project/rii/) and
[finding documents by words](https://pypi.org/project/nltk/). Outside of Python, ElasticSearch is a popular inverted
index search tool. Each of these has goals outside of FilterBox's niche; there are no plans to expand FilterBox towards
these functions.

____
