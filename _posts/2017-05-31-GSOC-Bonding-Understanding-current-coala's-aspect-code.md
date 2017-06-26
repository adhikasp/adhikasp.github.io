---
title: "GSOC Bonding - Understanding current coala's aspect code"
layout: single
excerpt: "This post is part of my GSOC journey in coala. Even though aspect
project in coala still in its infancy, there are already 
exist a basic prototype of itself in coala codebase. Today, I'll dive into
these code and try to analyze the core part about aspect."
sitemap: true
author_profile: false
category: "developer"
tags:
  - "English"
  - "coala"
---

This post is part of my GSOC journey in coala.

Even though aspect project in coala still in its infancy, there are already 
exist a basic prototype of itself in coala codebase. Today, I'll dive into
these code and try to analyze the core part about aspect.

> Seek First to Understand, then to be Understood

That is a quote from the [7 habbit of Stephen Covey](https://www.stephencovey.com/7habits/7habits-habit5.php).
It is a habbit for social interaction, but idk, I'll try to make some connection
here because I feel like it. It is important that I understand how the aspect
code work first, and then iI could improve it with confidence and get some idea
on how to code, the style, and the design flow.

## Overview

As of today, commit [`efa6727`](https://github.com/coala/coala/tree/efa6727f61f7aa07b4cf38e4ddb34318a978bb62),
coala have the the basic idea of what is an "aspect" is and a small aspect tree.
All of this code can be found in `coalib.bearlib.aspects` module. Beside this,
there also a small aspect code usage in `coalib.bears` class that used to bind
a aspect into `Bear` class. And also,  according to coala strict 100% code 
coverage, we have plenty of test to make sure the aspect code is healthy and 
working.

Let's start with the overviews of files under `coalib.bearlib.aspects` module.

- `__init.py__` - "loaders" function to access all aspect class.
- `base.py` - parent class of aspect that define what an aspect is 
- `meta.py` - aspect metaclass that has function to relate between aspects
- `taste.py` - define taste class, kinda like an "attribute" of a class
- `collections.py` - special collection object for aspect (I WROTE THIS :D)
- `docs.py` - define documentation object (like metadata) that each aspect 
              should have
- `root.py`, `Metadata.py`, `Redundacy.py`, `Spelling.py` - the definition of
  aspect

### Glossary

Before going further, I will explain some of few keyterm used.

- aspect - the class itself
- aspectinstance - instance object of a taste. Always have a language.
- taste - a measurable properties that help define an aspect. A taste could 
  have 1 or more language associated with it.
- `Language` class -  provide data structure for a (programming) language 
  information.

### Defining aspect in `base.py`

Next, let's take a look first on *how aspect defined* inside `base.py`.

```python
class aspectbase:

    def __init__(self, language, **taste_values):
        # bypass self.__setattr__
        self.__dict__['language'] = Language[language]
        for name, taste in type(self).tastes.items():
            if taste.languages and language not in taste.languages:
                if name in taste_values:
                    raise TasteError('%s.%s is not available for %s.' % (
                        type(self).__qualname__, name, language))
            else:
                setattr(self, name, taste_values.get(name, taste.default))

    def __eq__(self, other):
        return type(self) is type(other) and self.tastes == other.tastes

    @property
    def tastes(self):
        return {name: self.__dict__[name] for name in type(self).tastes
                if name in self.__dict__}

    def __setattr__(self, name, value):
        if name not in type(self).tastes:
            raise AttributeError(
                "can't set attributes of aspectclass instances")
        super().__setattr__(name, value)
```

Class `aspectbase` serve as the parent class of every aspect in coala. From 
reading the `__init__` method, we know that aspect has 2 type of "attribute",
which is language and tastes. An aspect's language attribute is a Language 
object taken from coalang (`coalib.bearlib.languages`) modules. 

After that, the taste attribute is initialized.

```python
for name, taste in type(self).tastes.items():
    if ...:
        # skipped
    else:
        setattr(self, name, taste_values.get(name, taste.default))
```

First, in the else section, all the taste from an aspect definition will be 
initialized to the aspectinstance. If we define a custom value for a taste, that 
value will override the default one.

```python
if taste.languages and language not in taste.languages:
    if name in taste_values:
        raise TasteError('%s.%s is not available for %s.' % (
            type(self).__qualname__, name, language))
```

The if part above is handling the case when an aspectinstance was initialized 
with a custom taste value under the language that the taste itself doesn't 
support. In that case, the whole initialization will raise a `TasteError` 
exception. 

In short, we could instance an aspect like this:

```python
>>> LineLength('Python')
<....Root.Formatting.LineLength object at 0x...>

>>> LineLength('Python').tastes
{'max_line_length': 80}

>>> LineLength('Python', max_line_length=100).tastes
{'max_line_length': 100}
```

Let's move on to the next function, shall we.

```python
def __eq__(self, other):
    return type(self) is type(other) and self.tastes == other.tastes
```

Overriding the `===` operator to compare the `type` object, make sure it has
the same metaclass `aspectclass` (more on this metaclass later), and have the
same tastes set.

```python
@property
def tastes(self):
    return {name: self.__dict__[name] for name in type(self).tastes
        if name in self.__dict__}
```

Binding all taste into a single callable dict object for convience. So the 
`LineLength('Python').tastes` will be possible. The `@property` decorator make
the `tastes` attribute kind of a readonly attribute of an aspect class.

```python
def __setattr__(self, name, value):
if name not in type(self).tastes:
    raise AttributeError(
        "can't set attributes of aspectclass instances")
super().__setattr__(name, value)
```

Block any attempt to add another taste object into this after instatiation.
The tastes list is kind of immutable.


### Inter aspect relation in `meta.py`

Aspect is structured like a tree. Last month, I wrote a small script that 
traverse these tree and generate the tree diagram.

![aspect tree diagram](https://raw.githubusercontent.com/adhikasp/aspect-tree-diagram/master/aspect_diagram.png)

The `meta.py` hold the metaclass declaration that responsible to link a parent
aspect and it's children aspect, recursively. Also, this metaclass hold a clever
function/decorator to automatically handle this linking process when declaring 
the children aspect. Let's take a look at the code.

```python
class aspectclass(type):
    def __init__(cls, clsname, bases, clsattrs):
        """
        Initializes the ``.subaspects`` dict on new aspectclasses.
        """
        cls.subaspects = {}
```

Each aspect should have dictionary called `subaspects` that hold its children.

```python
    @property
    def tastes(cls):
        """
        Get a dictionary of all taste names mapped to their
        :class:`coalib.bearlib.aspects.Taste` instances.
        """
        if cls.parent:
            return dict(cls.parent.tastes, **cls._tastes)

        return dict(cls._tastes)
```

Define a property to access tastes. The `cls._tastes` attribute was defined
manually in the `Root` aspect as an empty dict. Later, the children aspect will
fill their `_tastes` with their own tastes and the parent's tastes. More on this
later. Note that the `tastes` function on `aspectbase` was actually refering to
this function for the implementation detail. This was the actual method used
when we call `LineLength('Python').tastes`.

```python
    def subaspect(cls, subcls):
        """
        The sub-aspectclass decorator.

        See :class:`coalib.bearlib.aspects.Root` for description
        and usage.
        """
        aspectname = subcls.__name__

        docs = getattr(subcls, 'docs', None)
        aspectdocs = Documentation(subcls.__doc__, **{
            attr: getattr(docs, attr, '') for attr in
            list(signature(Documentation).parameters.keys())[1:]})

        # search for tastes in the sub-aspectclass
        subtastes = {}
        for name, member in getmembers(subcls):
            if isinstance(member, Taste):
                # tell the taste its own name
                member.name = name
                subtastes[name] = member

        class Sub(subcls, aspectbase, metaclass=aspectclass):
            __module__ = subcls.__module__

            parent = cls

            docs = aspectdocs
            _tastes = subtastes

        members = sorted(Sub.tastes)
        if members:
            Sub = generate_repr(*members)(Sub)

        Sub.__name__ = aspectname
        Sub.__qualname__ = '%s.%s' % (cls.__qualname__, aspectname)
        cls.subaspects[aspectname] = Sub
        setattr(cls, aspectname, Sub)
        return Sub
```

This not-so-simple `subaspect` method is the decorator used to declare and link
a children aspect to its parent. I will describe this method part by part.

```python
aspectname = subcls.__name__
```

Save the children aspect name for further reference.

```python
docs = getattr(subcls, 'docs', None)
aspectdocs = Documentation(subcls.__doc__, **{
    attr: getattr(docs, attr, '') for attr in
    list(signature(Documentation).parameters.keys())[1:]})
```

Setup the Documentation object of the aspect children.

```python
subtastes = {}
for name, member in getmembers(subcls):
    if isinstance(member, Taste):
        # tell the taste its own name
        member.name = name
        subtastes[name] = member
```

Search and collect every Taste object that declared on the aspect children
class and append it to `subtastes` dict.

```python
class Sub(subcls, aspectbase, metaclass=aspectclass):
    __module__ = subcls.__module__

    parent = cls

    docs = aspectdocs
    _tastes = subtastes
```

Declaring class to serve as "prototype" object that will override the aspect
children class.

```python
members = sorted(Sub.tastes)
if members:
    Sub = generate_repr(*members)(Sub)
```

Just auto generating the tastes string representation.

```python
Sub.__name__ = aspectname
Sub.__qualname__ = '%s.%s' % (cls.__qualname__, aspectname)
```

Declaring children aspect shortname as `__name__` and its fully qualified name
that contain its parent name too as `__qualname__`.

```python
cls.subaspects[aspectname] = Sub
```

Reference back the children aspect on its parent aspect.

```python
return Sub
```

Return the modified class. So this function could be used as decorator for a 
class.


## Summary

The aspect in coala was defined on 2 part, its base class `aspectbase` serve
as the foundation that define its basic characteristic and how we can declare
a basic aspect. Next we have the metaclass `aspectclass` that responsible for
parent-children relationship of an aspect and provide utility function to
declare new child aspect.
