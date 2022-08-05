# HashBox

Container for finding Python objects by matching attributes. 

Uses hash-based methods for storage and retrieval, so find is very fast.

[Finding objects using HashBox can be 5-10x faster than SQLite.](https://github.com/manimino/hashbox/blob/main/examples/perf_demo.ipynb)

```
pip install hashbox
```

[![tests Actions Status](https://github.com/manimino/hashbox/workflows/tests/badge.svg)](https://github.com/manimino/hashbox/actions)
[![Coverage - 100%](https://img.shields.io/static/v1?label=Coverage&message=100%&color=2ea44f)](test/cov.txt)
[![license - MIT](https://img.shields.io/static/v1?label=license&message=MIT&color=2ea44f)](/LICENSE)
![python - 3.7+](https://img.shields.io/static/v1?label=python&message=3.7%2B&color=2ea44f)


### Usage:

```
from hashbox import HashBox
hb = HashBox(                                # Make a HashBox
    [{'color': 'green', 'type': 'apple'},    
    {'color': 'green', 'type': 'frog'}]      # Containing any type of objects
    on=['color', 'type'])                    # Define attributes to find by
hb.find({'color': 'green', 'type': 'frog'})  # Find by attribute match
```

The objects can be anything: class instances, namedtuples, dicts, strings, floats, ints, etc.

There are two classes available.
 - HashBox: can `add()` and `remove()` objects. 
 - FrozenHashBox: faster finds, lower memory usage, and immutable. 

## Examples

Expand for sample code.

<details>
<summary>Match and exclude multiple values</summary>
<br>


```
from hashbox import HashBox

objects = [
    {'item': 1, 'size': 10, 'flavor': 'melon'}, 
    {'item': 2, 'size': 10, 'flavor': 'lychee'}, 
    {'item': 3, 'size': 20, 'flavor': 'peach'},
    {'item': 4, 'size': 30, 'flavor': 'apple'}
]

hb = HashBox(objects, on=['size', 'flavor'])

hb.find(
    match={'size': [10, 20]},                # match anything with size in [10, 20] 
    exclude={'flavor': ['lychee', 'peach']}  # where flavor is not in ['lychee', 'peach']
)  
# result: [{'item': 1, 'size': 10, 'flavor': 'melon'}]
```
</details>

<details>
<summary>Accessing nested attributes</summary>
<br />
Attributes can be functions. Function attributes are used to get values from nested data structures.

```
from hashbox import HashBox

objs = [
    {'a': {'b': [1, 2, 3]}},
    {'a': {'b': [4, 5, 6]}}
]

def get_nested(obj):
    return obj['a']['b'][0]

hb = HashBox(objs, [get_nested])
hb.find({get_nested: 4})  
# result: {'a': {'b': [4, 5, 6]}}
```
</details>

<details>
<summary>Derived attributes</summary>
<br />
Function attributes are very powerful. Here we find string objects with certain characteristics.

```
from hashbox import FrozenHashBox

objects = ['mushrooms', 'peppers', 'onions']

def o_count(obj):
    return obj.count('o')

f = FrozenHashBox(objects, [o_count, len])
f.find({len: 6})       # returns ['onions']
f.find({o_count: 2})   # returns ['mushrooms', 'onions']
```
</details>

<details>
<summary>Greater than, less than</summary>
<br />
HashBox and FrozenHashBox have a function <code>get_values(attr)</code> which gets the set of unique values
for an attribute. 

Here's how to use that to find objects having <code>x >= 3</code>.
```
from hashbox import HashBox

data = [{'x': i} for i in [1, 1, 2, 3, 5]]
hb = HashBox(data, ['x'])
vals = hb.get_values('x')                # get the set of unique values: {1, 2, 3, 5}
big_vals = [x for x in vals if x >= 3]   # big_vals is [3, 5]
hb.find({'x': big_vals})                 # result: [{'x': 3}, {'x': 5}
```
</details>

<details>
<summary>Handling missing attributes</summary>

- Objects that are missing an attribute will not be stored under that attribute. This saves lots of memory.
- To find all objects that have an attribute, match the special value <code>ANY</code>. 
- To find objects missing the attribute, exclude <code>ANY</code>.
- In functions, raise MissingAttribute to tell HashBox the object is missing.

```
from hashbox import HashBox, ANY
from hashbox.exceptions import MissingAttribute

def get_a(obj):
    try:
        return obj['a']
    except KeyError:
        raise MissingAttribute  # tell HashBox this attribute is missing

objs = [{'a': 1}, {'a': 2}, {}]
hb = HashBox(objs, ['a', get_a])

hb.find({'a': ANY})          # result: [{'a': 1}, {'a': 2}]
hb.find({get_a: ANY})        # result: [{'a': 1}, {'a': 2}]
hb.find(exclude={'a': ANY})  # result: [{}]
```
</details>

### Recipes
 
 - [Auto-updating](https://github.com/manimino/hashbox/blob/main/examples/update.py) - Keep HashBox updated when attribute values change
 - [Wordle solver](https://github.com/manimino/hashbox/blob/main/examples/wordle.ipynb) - Demonstrates using `functools.partials` to make attribute functions
 - [Collision detection](https://github.com/manimino/hashbox/blob/main/examples/collision.py) - Find objects based on type and proximity (grid-based)
 - [Percentiles](https://github.com/manimino/hashbox/blob/main/examples/percentile.py) - Find by percentile (median, p99, etc.)

### API documentation:
 - [HashBox](https://hashbox.readthedocs.io/en/latest/hashbox.mutable.html#hashbox.mutable.main.HashBox)
 - [FrozenHashBox](https://hashbox.readthedocs.io/en/latest/hashbox.frozen.html#hashbox.frozen.main.FrozenHashBox)

____

## How it works

For every attribute in HashBox, it holds a dict that maps each unique value to the set of objects with that value. 

HashBox is roughly this: 
```
HashBox = {
    'attribute1': {val1: {objs}, val2: {more_objs}},
    'attribute2': {val3: {objs}, val4: {more_objs}}
}
```

During `find()`, the object sets matching the query values are retrieved, and set operations like `union`, 
`intersect`, and `difference` are applied to get the final result.

That's a simplified version; for way more detail, See the "how it 
works" pages for [HashBox](hashbox/mutable/how_it_works.md) and [FrozenHashBox](hashbox/frozen/how_it_works.md).

### Related projects

HashBox is a type of inverted index. It is optimized for its goal of finding in-memory Python objects.

Other Python inverted index implementations are aimed at things like [vector search](https://pypi.org/project/rii/) and
[finding documents by words](https://pypi.org/project/nltk/). Outside of Python, ElasticSearch is a popular inverted
index search tool. Each of these has goals outside of HashBox's niche; there are no plans to expand HashBox towards
these functions.

____

<div align="center">
<img src="https://github.com/manimino/hashbox/blob/main/docs/hashbox-logo.png"><br>
</div>
