# Dictionaries

* [Item 1](): `dict` vs. `class`
* [Item 2](): Use `operator.itemgetter` instead of `__getitem__`
* [Item 3](): Use `itemgetter` only for one context
* [Item 4](): Get item or default
* [Item 5](): Walking trough nested dictionaries
* [Item 6](): Complex operations on dictionaries

## Item 1: `dict` vs. `class`

Dictionaries can be integrated very easy to your code and offers a great flexibility (e.g. as raw input from tools). But as great as the advantages of dictionaries are, they have their downsides. Source code that implements dictionary-based data flows can become very complex and confusing (to understand), which makes in worst case maintenance and (required) extensions impossible. 

If you're an experienced Python3 programmer, then you certainly know strengths of dictionaries and be aware of its shortcomings. Otherwise, it can be a code choice to implement your logic in the classic object-oriented way for the beginning. 

## Item 2: Use `operator.itemgetter` instead of `__getitem__`

The easiest way to obtain a value out of a `dict` is to use the `__getitem__` method.

```python
>>> finding = {"weakness": "SQL Injection"}
>>> finding["weakness"]               # Calls __getitem__("weakness")
SQL Injection
>>> finding.__getitem__("weakness") 
SQL Injection
```

### Short Comings

Usually, when working with instances of `dict`, the method `__getitem__` is quite fine - simple and does its job. But for more complex scenarios the method very quickly reaches its limits:

- Every call of `__getitem__` expires a `key` argument. Even in small data flows, this creates redundancy and can lead to clutter as well as visual noise.
- Sometimes when working with `dict` objects, it's more elegant to read more than one value at once, but the method `__getitem__` can only return one value per operation.
- The method `__getitem__` don't support returning a default value, if the given key doesn't exists within a `dict` (see Recipe X: Get item or default)

### Solution
Definition of a getter method
```python
>>> import operator
>>> weakness = operator.itemgetter("weakness")
>>> weakness
operator.itemgetter('weakness')
```
Usage of a getter method
```python
>>> finding1 = {"weakness": "SQL Injection", "title": "Product search function"}
>>> finding2 = {"weakness": "Cross-Site Scripting", "title": "Contact form"}
>>> weakness(finding1)
'SQL Injection'
>>> weakness(finding2)
'Cross-Site Scripting'
```
A further advantage of `operator.itemgetter` is the initialisation of  getter methods, which read multiple values. If multiple values are read, the getter method returns a `tuple` containing the corresponding values.
``` python
>>> operator.itemgetter("weakness", "title")(finding1)
('SQL Injection', 'Product search function')
```
**Note** that the values within the returned tuple are only shallow copies. If an object is mutable, all changes are made on the original instance of the object and can lead to unexpected behavoir of the application.

### Conclusion
Getter methods bring some advantages compared to `__getitem__`:
- Getter methods can be reused. Therefore the `key` must only be defined once.
- If the `key` in the data model is changed, only one location in the code must be updated. Even if the name of the getter method is  misleading from now on (until the next refactoring process), the functionality behind the getter method is still working.
- Refactoring a method is much easier with support of an IDE, than find and replace all occurrences of magic strings.
- Getter methods can read multiple values at the same time. This feature is a elegant way for complex operations (e.g. see "Recipe 4: Sort and grouping") and creating views. 

## Item 3: Use `itemgetter` only for one context

In software architecture, this is known as "separation of concerns". And this principle has a very serious reason. Mixing responsibilities can result in unresolvable dependencies among objects, methods, and modules when code is modified, extended, or maintained. Therefore, it is very important to have such situations and dependencies in mind, because afterwards it is often too late and the damage is done.

### Problem
Modifying a dictionary is relatively easy. Only keys or structural elements have to be changed. If objects are loaded from XML, YAML or JSON, for example, no modifications are necessary because the changed object is simply deserialized in its current shape. But for the logic of the data flow, it already looks quite different.

 The following example already shows the downside of dictionaries in a small dimension: One getter method is responsible for two different models (report and finding).
```python
>>> import operator
>>> report = {"title":"Sample Customer Test"}
>>> finding = {"title":"Server-side misconfiguration for HTTP Security Header"}
>>> title = operator.itemgetter("title")
>>> title(report)
'Sample Customer Test'
>>> title(finding)
'Misconfigured HTTP Security Header'
```

Imagine after a refactoring process the "title" in model "report" is renamed to "heading".

```python
>>> report = {"heading":"Sample Customer Test"}
>>> title(report)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'title'
```
The problem is obvious. Also when refactoring the getter method `title([dict])`, the problem can't be solved, because the keys are different.

### Solution
As a "quick" solution a further getter method is created to handle the changed key name. As  result each model has its own getter method and the data flow works.
```python
>>> import operator
>>> report = {"heading":"Sample Customer Test"}
>>> finding = {"title":"Server-side misconfiguration for HTTP Security Header"}
>>> report_title = operator.itemgetter("heading")
>>> finding_title = operator.itemgetter("title")
>>> report_title(report)
'Sample Customer Test'
>>> finding_title(finding)
'Misconfigured HTTP Security Header'
```
Why is this only a quick solution? Imagine the data model contains 10, 15, 20, 50 or more keys (e.g. when reading the raw output of tools) - some keys always are existing, some keys are only optional, other ones also contain nested sub-dictionaries. 

The above shown approach already shows a general problem of dictionaries: encapsulation of data - and the loosy approach to build a way around. If such situations occur more often, especially when simple getter methods are created and used, then a class-based implementation should be preferred. Dozens of "free-hanging" getter methods are not more flexible than class implementations. The opposite is true: In the end it is not clear what the getter methods are responsible for and a structured naming convention does not really help.

### Conclusion

The problem can be solved only after it is clear that a task can be covered by a dictionary. The dictionary itself is not the problem, but the code complexity as well as the effort of changes to the logic of the data flow, which process the dictionary (see Receipe 1). Otherwise, you're digging your own grave, code line by code line, helper function by helper function.

## Item 4: Get item or default

For optional fields, a dict-based implementation can quickly lead to unwanted behavior. If fields do not necessarily have to exist, the data flow needs a sufficient implementation to handle such circumstances. A simple solution is returning a default value, which is used to replace the non-existence of a value with an empty or standard one. Default values are a great help to avoid rasing errors that interrupt the data flow.

### Problem

`operator.itemgetter` doesn't support default values for not existing `key` arguments.

```python
>>> import operator
>>> finding = {"title":"Misconfigured HTTP Security Header"}
>>> operator.itemgetter("severity")(finding)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'severity'
```
The class `dict` offers the method `dict.get(key, default=None)`, but causing some noise and redundancy (see "Recipe 2: Use `operator.itemgetter` instead of `__getitem__`")
```python
>>> finding = {"title":"Misconfigured HTTP Security Header"}
>>> finding.get("severity", "Unknown")
'Unknown'
```
### Solution
A simple solution is a wrapper method to `operator.itemgetter`:
```python
def default_itemgetter(item, *items, default=None):
    """
    Wrapper method to operator.itemgetter with support for a default
    value, if an index or key doesn't exist.
    """
    def wrapped(obj):
        try:
            return operator.itemgetter(item, *items)(obj)
        except IndexError:
            return default
        except KeyError:
            return default

    return wrapped
```
### Examples
```python
>>> raw_result = {"rating": "Medium"}
>>> raw_result2 = {}
>>> rating = default_itemgetter("rating",default="Unknown")
>>> rating(raw_result)
Medium
>>> rating(raw_result2)
Unknown
>>> default_itemgetter("rating")(raw_result)
Medium
>>> default_itemgetter("score",default=0.0)(raw_result)
0.0
```

### Conclusion
Using default values can reduce code complexity and simplify the implementation of the data flow, because unnecessary checks are omitted.Default values allow the implementation to continue in the normal workflow. If necessary, extensions to existing methods are required.

## Item 5: Walking trough nested dictionaries

Another disadvantage of dictionaries becomes clear when working with nested `dict` objects. The necessary code complexity for reading, processing and error prevention can increase very quickly.

### Problem

Working with nested dictionaries can produce cluttered and error-prone code - especially if items within the data model are optoinal. 
```python
>>> finding = {"severity":{"cvss":{"rating":"Medium"}}}
>>> finding["severity"]["cvss"]["rating"]
'Medium'
>>> finding["severity"]["cvss"]["score"]
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'score'
>>> >>> finding["severity"]["simple"]["rating"]
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'simple'
```
Quick shot to avoid rasing `KeyError`
```python
>>> finding.get("severity", {}).get("cvss", {}).get("score",0.0)
0.0
>>> finding.get("severity", {}).get("rating", "Medium")
'Medium'
```

### Solution

An elegant solution is to outsource the traversal of a nested dictionary to a helper function that expects a path. The path represents the order of keys to reach the corresponding item.

```python
def nested_itemgetter(path, sep = "/", default = None):
    """
    Returns an equivalent to "operator.itemgetter" with the ability to walk
    along a given path through a nested dictionary. If a visited key doesn't
    exist "default" is returned.
    """
    def walk(item):
        try:
            return functools.reduce(lambda item, key: item[key], path.split(sep), item)
        except KeyError:
            return default

    return walk
```

In order that the function works correctly, the following conditions must be ensured:
- All keys along the walked path must be strings
- All visited items while walking trough the nested dictionary must implements `__getitem__` (e.g. like datatype `dict`). Otherwise an `TypeError` will be raised.

The approach shown above only reads one value - but can be extended to support reading multiple paths. If reading more than two or three values from a sub-dictionary, than it's less complex to read the hole sub-dictionary and working on it directly.

### Examples

```python
>>> raw_result= {"severity":{"cvss":{"rating":"Medium"}}}
>>> rating = nested_itemgetter("severity/cvss/rating")
>>> rating(raw_result)
Medium
>>> nested_itemgetter("severity.cvss.rating",sep=".")(raw_result)
Medium
>>> nested_itemgetter("severity/cvss/score", default=0.0)(raw_result)
0.0
>>> nested_itemgetter("severity/simple/rating", default="Unknown")(raw_result)
Unknown
```
### Conclusion

Python offers an elegant way to solve this problem. However, it should also be said at this point that, as already discussed, it is a question of the requirements to your data flow implementation. If a `dict` is the wrong choice, also no elegant helper function can change that fact.



## Item 6: Complex operations on dictionaries
**Problem**

How to sort and group a list of of dictionaries effectively?

**Example 1**

Sorts a list of findings by "weakness" and "title" in ascending order - Grouping is indirectly done when sorting by "weakness"
```python
>>> import operator
>>> finding1 = {"weakness": "Information Disclosure", "title": "HTTP 'X-Powered-By' Header"}
>>> finding2 = {"weakness": "SQL Injection", "title": "Product search function"}
>>> finding3 = {"weakness": "Cross-Site Scripting", "title": "Contact form"}
>>> finding4 = {"weakness": "Information Disclosure", "title": "HTTP 'Server' Header"}
>>> findings = [finding1, finding2, finding3, finding4]
>>> [print(finding) for finding in sorted(findings, key=operator.itemgetter("weakness","title"))]
{'weakness': 'Cross-Site Scripting', 'title': 'Contact form'}
{'weakness': 'Information Disclosure', 'title': "HTTP 'Server' Header"}
{'weakness': 'Information Disclosure', 'title': "HTTP 'X-Powered-By' Header"}
{'weakness': 'SQL Injection', 'title': 'Product search function'}
```

**Example 2**

- Findings are grouped by "weakness".
- Groups are sorted descending by highest severity within the group.
- Groups with same highest "severity" are sorted ascending by "title".
- Findings within a group are also sorted descending by "severity".
- Findings with same "severity" within a group are sorted ascending by "title", too.

```python
def helper_function(iterable):
    """
    Returns a dict with grouped findings and sorted by severity and title
    """
    result = defaultdict(list)

    # Broken Recipe 1 to make the example easy to understand
    # This example also shows the limits of operator.itemgetter,e.g. when
    # modifying the sort order by adding '-' for reversing
    for finding in sorted(iterable, key=lambda f: (-f["severity"].value, f["weakness"], f["title"])):
        result[finding["weakness"]].append(finding)

    return result

[...]

from pyfindings.rating import Rating

finding1 = {"weakness": "Information Disclosure", "title": "HTTP 'X-Powered-By' Header",
                     "severity": Rating.NONE}
finding2 = {"weakness": "SQL Injection", "title": "Product search function", "severity": Rating.HIGH}
finding3 = {"weakness": "Cross-Site Scripting", "title": "Contact form", "severity": Rating.HIGH}
finding4 = {"weakness": "Information Disclosure", "title": "HTTP 'Server' Header", "severity": Rating.NONE}
result = helper_function([finding1, finding2, finding3, finding4])

for group in result.keys():
    print(f"{group} - ({len(result[group])} Issue(s))")
    for finding in result[group]:
        print(f"\t{finding['weakness']} - {finding['title']} ({finding['severity']})")
```
Output:
```
Cross-Site Scripting - (1 Issue(s))
	Cross-Site Scripting - Contact form (High)
SQL Injection - (1 Issue(s))
	SQL Injection - Product search function (High)
Information Disclosure - (2 Issue(s))
	Information Disclosure - HTTP 'Server' Header (None)
	Information Disclosure - HTTP 'X-Powered-By' Header (None)
```