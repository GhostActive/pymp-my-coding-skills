# Dictionaries

* [Item 1](#item-1-dict-vs-class): `dict` vs. `class`
* [Item 2](#item-2-use-operatoritemgetter-instead-of-__getitem__): Use `operator.itemgetter` instead of `__getitem__`
* [Item 3](#item-3-use-itemgetter-only-for-one-context): Use `itemgetter` only for one context
* [Item 4](#item-4-get-item-or-default): Get item or default
* [Item 5](#item-5-walking-trough-nested-dictionaries): Walking trough nested dictionaries
* [Item 6](#item-6-complex-operations-on-dictionaries): Complex operations on dictionaries

## Item 1: `dict` vs. `class`

When are dictionaries better than classes and vice versa? The programming paradigma used is the key. Both concepts can represent data structures: Classes via attributes and properties, dictionaries via key-value pairs. Classes, however, have all the advantages of object-oriented programming, e.g. encapsulation. Dictionaries are efficient for functional programming due to their simplicity.

At first view, classes look like more effort for implementation - evil trap and a popular beginner's mistake. Dictionaries just need the effort in a different way. Dealing with dictionaries can very quickly lead to tragedy, for example, due to the lack of encapsulation and ability of inheritance of contained data structures. As a simple Rule: Classes bring more stability and dictionaries more flexibility. However, the gained flexibility must be protected by a robust implementation that processes the dictionaries. And that's where the trap can catches you and the effort can suddenly become overwhelming by implementing dozens of *free-haning* validation methods, helper functions and so on. As a beginner, this is often difficult to see and correctly calculate due to lack of experience.

### Conclusion

The use of `dict` or `class` depends on the programming paradigm that is used. In practice, however, programming skills are more important. Dictionaries are efficient if you understand functional programming with Python. Otherwise, the implementation can become very complex, confusing or even become unwillingly an imitation of object-oriented programming. If you are a beginner or generally have little experience in functional programming, prefer a class-based implementation, even if dictionaries are more efficient, to avoid structural mistakes and bad smells. The structure of classes may mean additional effort, but it brings more stability.

## Item 2: Use `operator.itemgetter` instead of `__getitem__`

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
- The method `__getitem__` don't support returning a default value, if the given key doesn't exists within a `dict` a `KeyError` is raised (see *[Item 4](#item-4-get-item-or-default): Get item or default*)
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
```
**Note** that the values within the returned `tuple` are only shallow copies. If an contained object is mutable, all changes are made on the original instance of the object, too.

### Conclusion
Getter methods bring some advantages compared to `__getitem__`:
- Getter methods can be reused. Therefore the `key` must only be defined once.
- If the `key` in the data model is changed, only one location in the code must be updated. Even if the name of the getter method is  misleading from now on (until the next refactoring process), the functionality behind the getter method is still working.
- Getter methods can read multiple values per operation. This feature is sometimes an elegant way for complex operations (see *[Item 6](#item-6-complex-operations-on-dictionaries): Complex operations on dictionaries*). 

## Item 3: Use `operator.itemgetter` only for one context

Mixing responsibilities is very dangarous and can result in large efforts when modifying or extending source code.

### Problem
The following example demonstrates this effect in a very small dimension. One getter method is responsible for two different data models (article and reports).
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
Imagine after a refactoring process the "title" in model "article" is renamed to "heading".

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
>>> report_title = operator.itemgetter("title")
>>> article_title(article)
'Mastering Python: A Quick Guide'
>>> report_title(report)
' Refactoring Code
```
The problem is solved, but another one arises: The code base becomes a bit more cluttered. What happens after the 10th, 20th or 50th modification of this kind? 

## Conclusion

Don't mix responsibilities to avoid unwanted dependencies. If such situations occur more often, especially when simple getter methods are created and used, then a class-based implementation should be preferred for more stability. Dozens of *free-hanging* getter or helper functions imitating an object-oriented approach are not more flexible than a solid class implementation.

## Item 4: Get item or default

For optional fields, a `dict`-based implementation can quickly lead to unwanted behavior. If fields don't exist, the data flow needs a robust implementation to handle such circumstances. A simple solution is returning a default value, which is used to replace the non-existence.

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
The standard type `dict` offers the method `dict.get(key, default=None)` requiring a `key` argument (analog see *[Item 2](#item-2-use-operatoritemgetter-instead-of-__getitem__): Use `operator.itemgetter` instead of `__getitem__`*)
```python
>>> article.get("author", "Unknown")
'Unknown'
```
### Solution
A simple solution is using [collections.defaultdict](https://docs.python.org/3/library/collections.html#collections.defaultdict). Simple and fast if the default value is always of the same data type.
```python
>>> article = defaultdict(str,{"title": "Mastering Python: A Quick Guide"})
>>> article
defaultdict(<class 'str'>, {'title': 'Mastering Python: A Quick Guide'})
>>> article["title"]
'Mastering Python: A Quick Guide'
>>> article["author"]
''
```
A little tricky and error prone: If always `collections.defaultdict` is used, you can use `operator.itemgetter`.
```python
>>> import operator
>>> article = defaultdict(str,{"title": "Mastering Python: A Quick Guide"})
>>> operator.itemgetter("title","author")(article)
('Mastering Python: A Quick Guide', '')
```
A robust solution, for reading multiple values with flexible defaults, is to implement a simple wrapper method:
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
**Note** that the wrapper method, shown above, only supports keys of data type `str`. Furthermore, the value `obj` must be a `dict`-based object.
### Examples
```python
>>> article = {"title": "Mastering Python: A Quick Guide"}
>>> report = {"heading": "Refactor Code"}
>>> view = default_itemgetter(title= "Unknown", author= "Anonymous")
>>> view(article)
('Mastering Python: A Quick Guide', 'Anonymous')
>>> view(report)
('Unknown', 'Anonymous')
```

### Conclusion
Using default values can simpliy the code complexity, because validation checks are reduced to a certain extent. Regardless whether a value exists or not, the implementation can continue with workflow.

## Item 5: Walking trough nested dictionaries

Working with nested dictionaries can produce cluttered and error-prone code, especially if items within the data model are optoinal. The necessary code complexity for reading, processing and error prevention can increase very quickly.

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

An good solution is to create a helper function that expects a path of an nested item.

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
- All keys along the walked path must be strings
- All visited items while walking trough the nested dictionary must implements `__getitem__` (e.g. like datatype `dict`). Otherwise an `TypeError` will be raised.

Furthermore the approach only supports reading one value per operaion, but can be extended. If reading more than two or three values from a sub-dictionary, than it's less complex to read the hole sub-dictionary and working on it directly.

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

The example shows how a smart use of helper functions and some Python knowledge can improve code complexity and readability.  But if a `dict` is the wrong choice to meet the requirements, no helper function can change that fact.

## Item 6: Complex operations on dictionaries

This section shows by two examples with increasing complexity how nevertheless simple solutions can be implemented in Python.

**Samples**
Let's define the following task - The higher the number of priority, the more important the task.
```python
>>> task1 = {"category": "Work", "title": "Review and publish latest article", "priority": 4}
>>> task2 = {"category": "Work", "title": "Write article about Python", "priority": 3}
>>> task3 = {"category": "Entertainment", "title": "Watch a movie", "priority": 2}
>>> task4 = {"category": "Learning", "title": "Read a book about Python", "priority": 4}
```

How to sort and group a list of of dictionaries effectively?

**Example 1** - Simple sorting

* Group all tasks by *category*.
* Sort groups alphabetically ascending.
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
The grouping is done implicitly due to the sorting by category.

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
Learning (1)
	Read a book about Python (priority=4)
Work (2)
	Review and publish latest article (priority=4)
	Write article about Python (priority=3)
Entertainment (1)
	Watch a movie (priority=2)
```

## Conclusion
Despite the increasing complexity of the example, the complexity of the implementation is still quite simple. However, the solution requires some knowledge about Python and abstraction skills. The trick at this point is not to think of the requirements as a sequence of statements and implement them that way. Abstract thinking from the end is the key. Otherwise, the implementation becomes more complex.
