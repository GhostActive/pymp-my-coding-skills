# Clean Code

- Introduction

- Short Definition for "Clean Code"

- Further Reading: Clean Code in Python

## Item 1: Pythonic

## Item 2: Avoid unnecessary logic for control flow

## Item 3: Use default values

## Item 4: Implement *atomic* or *higher-ordered*

### Sample

```python
class Rating(Enum, metaclass=RatingMeta):
    UNKNOWN = ("Unknown", -1)
    NONE = ("None", 0)
    LOW = ("Low", 1)
    MEDIUM = ("Medium", 2)
    HIGH = ("High", 3)
    CRITICAL = ("Critical", 4)

    @property
    def name(self) -> str:
        return self._value_[0]

    ...
```

### Problem

An earlier version of the class contained a own variant of the `map` function intended to easily customize the item's label.

```python
def map(mapper):
    if not mapper:
        return self.name

    return mapper.get(self.name, self.name)
```

Often, the value of ratings is reported as *Informational* rather than *None*.

```python
>>> from rating import Rating
>>> rating = Rating.NONE
>>> rating
<Rating.NONE: 0>
>>> str(rating)
'None'
>>> mapper = {"None" : "Informational"}
>>> rating.map(mapper)
'Informational'
```

The above shown approach has various short-comings:

* The method `map` is designed with an valid intention, but only the item's label can be mapped. For the item's instance itself or property `value` a further mapping outside of the class is necessary - no consistent design.
* The parameter `mapper` is fixed to data type `dict`.

### Solution

The method `map` is neither *atomic* nor a *high-ordered function or operator*. In the final version of the class, the method was removed and replaced by simply invoking `dict.get(key,default)`.

```python
>>> mapper = {"None": "Informational"}
>>> rating = Rating.NONE
>>> mapper.get(rating.name, rating.name)
'Informational'
```

### Conclusion

The logic of a function or class should be closes and complete, but its use open. The user can then decide how to embed the logic into the current implementation. Keep this strategy in mind when designing and implementing functions or objects. There should be no restrictions or prescribed paths unless they are part of the logic to allow the most flexible use.

## Item 5: The use of design pattern


