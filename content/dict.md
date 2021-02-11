# Dictionaries

* [*Item 1*](#item-1-dict-vs-class): `dict` vs. `class`
* [*Item 2*](#item-2-prefer-operatoritemgetter-over-__getitem__): Prefer `operator.itemgetter` over `__getitem__`
* [*Item 3*](#item-3-use-operatoritemgetter-only-for-one-context): Use `operator.itemgetter` only for one context
* [*Item 4*](#item-4-get-item-or-default): Get item or default
* [*Item 5*](#item-5-walking-trough-nested-dictionaries): Walking trough nested dictionaries
* [*Item 6*](#item-6-complex-operations-on-dictionaries): Complex operations on dictionaries

## Item 1: `dict` vs. `class`

What is the better choice: classes or dictionaries? It depends on the programming paradigm used. Generally, both concepts can represent data structures. Classes via attributes and properties, dictionaries via key-value pairs. Classes, however, have all the advantages of object-oriented programming, e.g. encapsulation. Dictionaries are efficient for functional programming due to their simplicity.

At first view, classes seems to need more effort for implementation compared to dictionaries. Evil trap and a quickly overlooked pitfall. Dictionaries also needs effort, just in a different way. Dealing with dictionaries can very quickly lead to tragedy, for example, due to the lack of encapsulation and the missing ability of inheritance of the contained data structure. As a simple rule: Classes bring more structure, dictionaries more flexibility. However, the gained flexibility must be protected by a robust implementation processing the dictionaries. And that's the point where the trap can catches you, because the effort suddenly increases strongly, for example, by implementing dozens of *free-hanging* validation methods or helper functions. Especially as a beginner, this phenomenon is difficult to evaluate in its consequence.

To a certain degree, aspects of both paradigms can be integrated into the implementation. Nevertheless, there should be a basic decision which one is dominating. This decision influences the implementation of new features later. In a functional implementation, it is increasingly difficult to add object-oriented features and vice versa. That's the second point where the trap can catches you, when extending the code base.

### Conclusion

The use of `dict` or `class` depends on the programming paradigm. Practically, however, programming skills are more important then the theory behind. Dictionaries are efficient if you understand functional programming. Otherwise, the implementation can become very complex, confusing or even an imitation of object-oriented style. In this case use a class-based implementation.

## Item 2: Prefer `operator.itemgetter` over `__getitem__`

The easiest way to obtain a value out of a `dict` is to use the `__getitem__` method.

```python
>>> article = {"title": "Mastering Python: A Quick Guide"}
>>> article["title"]                        # Calls __getitem__("title")
'Mastering Python: A Quick Guide'
>>> article.__getitem__("title")
'Mastering Python: A Quick Guide'
```

### Short Comings

Normally, when working with instances of `dict`, the method `__getitem__` is quite fine - simple and does its job. But for more complex scenarios the method very quickly reaches its limits:

- Every call of `__getitem__` expires a `key` argument. Even within little code base, this creates redundancy that leads to clutter and visual noise.
- The method `__getitem__` don't support returning a default value, if the given key doesn't exist a `KeyError` is raised (see *[Item 4](#item-4-get-item-or-default): Get item or default*).
- Sometimes when working with `dict` objects, it's more elegant to read more than one value at once, but the method `__getitem__` can only return one value per operation (see *[Item 6](#item-6-complex-operations-on-dictionaries): Complex operations on dictionaries*).

### Solution

Definition of a getter method

```python
>>> import operator
>>> title = operator.itemgetter("title")
>>> title
operator.itemgetter('title')
```

Using a getter method

```python
>>> article1 = {"title": "Mastering Python: A Quick Guide"}
>>> article2 = {"title": "Dictionaries"}
>>> title(article1)
'Mastering Python: A Quick Guide'
>>> title(article2)
'Dictionaries'
```

A further advantage of `operator.itemgetter` is reading multiple values per operation. In this case the values are returned as `tuple`.

``` python
>>> article = {"title": "Mastering Python: A Quick Guide", "created":"2020-12-23", "content":"..."}
>>> operator.itemgetter("title", "created")(article)
('Mastering Python: A Quick Guide', '2020-12-23')
>>> keys = ["title", "created"]
>>> operator.itemgetter(*keys)(article)
('Mastering Python: A Quick Guide', '2020-12-23')
```

**Note** that the values within the returned `tuple` are only shallow copies. If a contained object is mutable, all changes affects the original instance of the object, which can lead to unexpected behavior.

### Example

The following examples shows, how to find same content over a list of specified keys in both variations. The use of `operator.itemgetter` looks a little more harmonic.

```python
# Helper function from https://docs.python.org/3/library/itertools.html#itertools-recipes
def all_equal(iterable, key=None):
    "Returns True if all the elements are equal to each other"
    g = itertools.groupby(iterable, key=key)
    return next(g, True) and not next(g, False)
```

```python
>>> article1 = {"title": "Mastering Python: A Quick Guide", "created": "2020-12-26", "status": "Final"}
>>> article2 = {"title": "Refactoring Python Code", "created": "2020-12-26", "status": "Draft"}
>>> article3 = {"title": "Functional Programming with Python", "created": "2020-12-26"}
>>> articles = [article1, article2, article3]
>>> relevant = ["title", "created"]
>>> [key for key in relevant if all_equal(articles, key=operator.itemgetter(key))]
['created']
>>> [key for key in relevant if all_equal(articles, key=lambda x: x[key])]
['created']
```
### Conclusion

Getter methods bring some advantages compared to `__getitem__`:
- Getter methods can be reused. Therefore the `key` must only be defined once.
- If the `key` in the data model is changed, only one location per getter method in the code must be updated. Even if the name of the getter method is  misleading by changes, the functionality behind the getter method is still working.
- Getter methods can read multiple values per operation. This feature is sometimes an elegant way for complex operations (see *[Item 6](#item-6-complex-operations-on-dictionaries): Complex operations on dictionaries*). 

However, if the use of `operator.itemgetter` tends to increase complexity unnecessarily, then `__getitem__` or `dict.get(key,default)` should be used.

## Item 3: Use `operator.itemgetter` only for one context

Mixing responsibilities is very dangerous and can result in large efforts when modifying or extending code afterwards.

### Problem

The following example demonstrates this effect in a very small dimension. One getter method is responsible for two different data models (articles and books).

```python
>>> import operator
>>> article = {"title": "Mastering Python: A Quick Guide"}
>>> book = {"title": "Refactoring Code"}
>>> title = operator.itemgetter("title")
>>> title(article)
'Mastering Python: A Quick Guide'
>>> title(book)
'Refactoring Code'
```

Imagine after a refactoring process the *title* in model *article* is renamed to *heading*.

```python
>>> article = {"heading": "Mastering Python: A Quick Guide"}
>>> title(article)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'title'
```

The problem is obvious. But also when refactoring the key within the getter method, the problem can't be solved, because the keys are different.

### Solution

A *quick* solution is to implement a further getter method handling the new key.

```python
>>> article_title = operator.itemgetter("heading")
>>> book_title = operator.itemgetter("title")
>>> article_title(article)
'Mastering Python: A Quick Guide'
>>> book_title(book)
'Refactoring Code'
```

The problem is solved, but another one arises: The source code becomes a bit more cluttered. What happens after the 10th, 20th or 50th modification of this kind?

### Conclusion

Don't mix responsibilities to avoid unwanted dependencies. Solving dependencies can lead to a great effort for refactoring the source code. Especially, if such situations occur more often, then it's an indicator for the imitation of object-oriented programming. Dozens of *free-hanging* getter, validation or helper functions are not less effort than a solid class implementation.

## Item 4: Get item or default

For optional fields, a `dict`-based implementation can quickly lead to unwanted behavior. If fields don't exist, the data flow needs a robust implementation to handle such circumstances. A simple solution is using a default value replacing the non-existence.

### Problem

`operator.itemgetter` doesn't support default values for not existing `key` arguments.

```python
>>> import operator
>>> article = {"title": "Mastering Python: A Quick Guide"}
>>> operator.itemgetter("author")(article)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'author'
```

The standard type `dict` offers the method `dict.get(key, default=None)` requiring a `key` argument and a further default value as optional parameter (analogue see *[Item 2](#item-2-prefer-operatoritemgetter-over-__getitem__): Prefer `operator.itemgetter` over `__getitem__`*).

```python
>>> article.get("author", "Unknown")
'Unknown'
```

### Solution

A simple solution is using [*collections.defaultdict*](https://docs.python.org/3/library/collections.html#collections.defaultdict), if the default value always is of the same data type.

```python
>>> article = defaultdict(str,{"title": "Mastering Python: A Quick Guide"})
>>> article
defaultdict(<class 'str'>, {'title': 'Mastering Python: A Quick Guide'})
>>> article["title"]
'Mastering Python: A Quick Guide'
>>> article["author"]
''
```

A less robust solution: If using `collections.defaultdict`, then `operator.itemgetter` doesn't raise errors

```python
>>> import operator
>>> article = defaultdict(str,{"title": "Mastering Python: A Quick Guide"})
>>> operator.itemgetter("title","author")(article)
('Mastering Python: A Quick Guide', '')
```

A robust solution, for reading multiple values with flexible default values can be reached by implementing a simple wrapper method:

```python
def default_itemgetter(**kwargs):
    """
    Wrapper method to operator.itemgetter supporting default values.
    """
    if kwargs is None:
        kwargs = dict()

    def wrapped(obj):
        return operator.itemgetter(*kwargs.keys())(
            {key: obj.get(key, kwargs[key]) for key in kwargs.keys()})

    return wrapped
```

**Note** that the wrapper method, shown above, only supports key arguments of data type `str`. Furthermore, the parameter `obj` must be a `dict`-based object.

### Examples

```python
>>> article = {"heading": "Mastering Python: A Quick Guide"}
>>> book = {"title": "Refactor Code"}
>>> view = default_itemgetter(heading= "Unknown", author= "Anonymous")
>>> view(article)
('Mastering Python: A Quick Guide', 'Anonymous')
>>> view(report)
('Unknown', 'Anonymous')
```

### Conclusion

Using default values can simplify the code complexity, because validation checks can be reduced. Regardless whether a value exists or not, the workflow can continue at runtime without raising errors or unexpected behavior.

## Item 5: Walking trough nested dictionaries

Working with nested dictionaries can produce cluttered and error-prone code, especially if items within the data model are optional. The necessary code complexity for reading, processing and error prevention can increase very quickly.

### Problem

Reading values from nested `dict` objects is error prone and creates clutter.

```python
>>> article = {"metadata":{"status":{"label":"Draft"}}}
>>> article["metadata"]["status"]["label"]
'Draft'
>>> article["metadata"]["status"]["id"]
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'id'
>>> article["metadata"]["tags"]
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'tags'
```

Quick shot to avoid `KeyError`

```python
>>> article.get("metadata", {}).get("status",{}).get("label","Unknown")
'Draft'
>>> article.get("metadata", {}).get("status",{}).get("id",0)
0
>>> article.get("metadata", {}).get("tags",[])
[]
```

### Solution

A good solution is to create a helper function that expects a *path* representing a nested item.

```python
def nested_itemgetter(path, sep = "/", default = None):
    """
    Getter function walking along the given path through a nested dictionary.
    If a visited key doesn't exist "default" is returned.
    """
    def walk(item):
        try:
            return functools.reduce(lambda item, key: item[key], path.split(sep), item)
        except KeyError:
            return default

    return walk
```

In order that the function, shown above, works correctly, the following conditions must be ensured:
- All keys along the walked path must be strings.
- All visited items while walking trough the nested dictionary must implements `__getitem__` (e.g. like datatype `dict`). Otherwise an `TypeError` will be raised.

Furthermore the approach only supports reading one value per operation, but can be extended. If reading more than two or three values from a sub-dictionary, than it's less complex to read the hole sub-dictionary and working on it directly.

### Examples

```python
>>> status = nested_itemgetter("metadata/status/label", default="Unknown")
>>> status(article)
'Draft'
>>> nested_itemgetter("metadata.status.label",sep=".", default="Unknown")(article)
'Draft'
>>> nested_itemgetter("metadata.status.id",sep=".", default=0)(article)
0
```

### Conclusion

The example shows how smart use of helper functions can improve code complexity and readability.  But if a `dict` is the wrong choice, no helper function can change that fact.

## Item 6: Complex operations on dictionaries

This section shows by two examples with increasing complexity how nevertheless simple solutions can be implemented in Python. The scenario is in each case: How to sort and group a list of of dictionaries effectively and with a minimum of effort?

**Samples**

Let's define the following tasks.

```python
>>> task1 = {"category": "Work", "title": "Review and publish latest article", "priority": 4}
>>> task2 = {"category": "Work", "title": "Write article about Python", "priority": 3}
>>> task3 = {"category": "Entertainment", "title": "Watch a movie", "priority": 2}
>>> task4 = {"category": "Learning", "title": "Read a book about Python", "priority": 4}
```

The higher the number for priority, the more important a task is.

**Example 1** - Simple sorting

* Group all tasks by *category*.
* Sort groups alphabetically in ascending order.
* Sort all items within a group alphabetically ascending by *title*.

```python
>>> import operator
>>> for task in sorted(tasks, key=operator.itemgetter("category","title")):
...     print(task)
... 
{'category': 'Entertainment', 'title': 'Watch a movie', 'priority': 2}
{'category': 'Learning', 'title': 'Read a book about Python', 'priority': 4}
{'category': 'Work', 'title': 'Review and publish latest article', 'priority': 4}
{'category': 'Work', 'title': 'Write article about Python', 'priority': 3}
```

The grouping is done implicitly due to the sorting by *category* and can continued, for example, with [*itertools.groupby*](https://docs.python.org/3/library/itertools.html#itertools.groupby) and `operator.itemgetter("category")` as parameter `key`.

**Example 2** - More complex sorting

* Group all tasks by *category*.
* Sort groups in descending order by *priority*. The *priority* of a group is the *priority* of the most important item of the group. Additionally, sort groups of the same *priority* alphabetically ascending.
* Tasks within a group are also sorted in descending order of *priority*. Tasks of the same *priority* are sorted alphabetically in ascending order by *title*.

First, define two helper functions that do the job:

```python
category = operator.itemgetter("category")
priority = operator.itemgetter("priority")
title = operator.itemgetter("title")

def prepare_tasks_for_print(iterable):
    result = defaultdict(list)

    for task in sorted(iterable, key=lambda t: (-priority(t), category(t), title(t))):
        result[category(task)].append(task)

    return result

def print_current_tasks(tasks):
    for group in tasks.keys():
        print(f"{group} ({len(tasks[group])})")

        for task in tasks[group]:
            print(f"\t{title(task)} (priority={priority(task)})")
```

Second, use the helper functions:

```python
>>> result = prepare_tasks_for_print([task1, task2, task3, task4])
>>> print_current_tasks(result)
```

Output:

```
Learning (1)
	Read a book about Python (priority=4)
Work (2)
	Review and publish latest article (priority=4)
	Write article about Python (priority=3)
Entertainment (1)
	Watch a movie (priority=2)
```

### Conclusion

Despite the increasing complexity of the examples, the complexity of the solutions are still quite simple. However, the solutions require some knowledge about Python and abstraction. The trick at this point is not to think of the requirements as a ordered sequence of statements for implementation. Abstract thinking from the end is the key. The [*functional programming modules*](https://docs.python.org/3/library/functional.html) can be useful for implementation including further recipes.
