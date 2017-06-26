---
title: "GSOC Phase 1 Late - Laying groundwork for practical aspects API"
layout: single
excerpt: "This time, I was laying groundwork to prepare practical aspects API
that coala could use in its core/processing part and in bears. Current aspect
module already have basic code to make themself *work*. But it still lacking
some essential API-like method to make aspects really usable on other part of 
coala."
sitemap: true
author_profile: false
category: "developer"
tags:
  - "English"
  - "coala"
---

This post is part of my GSOC journey in coala.

TLDR; I wrote these patches to coala:

- [AspectList: Follow class name convention](https://github.com/coala/coala/pull/4403)
- [AspectList: Overload `__init__` to accept strings](https://github.com/coala/coala/pull/4389)
- [Aspects: Create exception for aspects lookup](https://github.com/coala/coala/pull/4395)
- [AspectList: Add `get()` method](https://github.com/coala/coala/pull/4392)
- [aspectModule: Add `get()` method](https://github.com/coala/coala/pull/4412)


## What is this about?

Laying groundwork to prepare practical aspects API that coala could use in its
core/processing part and in bears. Current aspect module already have basic
code to make themself *work*. But it still lacking some essential API-like
method to make aspects really usable on other part of coala.

Let's take a look on each patch and see what it do.


### AspectList: Follow class name convention

<https://github.com/coala/coala/pull/4403>

This patch just rename aspectlist into AspectList to conform with PEP8 class
naming convention.


### AspectList: Overload `__init__` to accept strings

<https://github.com/coala/coala/pull/4389>

In here, I extends AspectList constructor method so we can create an AspectList
from list of string that hold aspect name.

Example:

```python
>>> aspectlist(['coalaCorrect', 'aspectsYEAH'])
[<aspectclass '...coalaCorrect'>, <aspectclass '...aspectsYEAH'>]
```

This is a pretty important feature to convert a list of specified aspect
that users write in configuration files to an AspectList.

Here is the code I write:

```python
def __init__(self, seq=()):
    """
    Initialize a new AspectList.

    >>> from .Metadata import CommitMessage
    >>> AspectList([CommitMessage.Shortlog, CommitMessage.Body])
    [<aspectclass '...Shortlog'>, <aspectclass '...Body'>]
    >>> AspectList(['Shortlog', 'CommitMessage.Body'])
    [<aspectclass '...Shortlog'>, <aspectclass '...Body'>]
    >>> AspectList([CommitMessage.Shortlog, 'CommitMessage.Body'])
    [<aspectclass '...Shortlog'>, <aspectclass '...Body'>]

    :param seq: A sequence containing either aspectclass, aspectclass
                instance, or string of partial/full qualified aspect name.
    """
    super().__init__((item if isaspect(item) else
                        coalib.bearlib.aspects[item] for item in seq))
```

The real change is only at the `super().__init__(...)` part. Before, we check
each item in `seq` to assert it was already an aspect object. But we change
that to only check if it is an aspect or not. If yes, then just pass the item
as is. But if it not (a string), we use aspects module search to search aspect
with given name and add it to new sequence.


### Aspects: Create exception for aspects lookup

<https://github.com/coala/coala/pull/4395>

I move aspect's exception to its own modules to avoid things like circular
import in the future. Then, I create bunch of LookupError derived errors to
replace current LookupError that was used by aspectModule.

But, I think I make a little mistake on designing the base AspectLookupError.
Here is the code.

```python
class AspectLookupError(LookupError):
    """
    Error raised when trying to search aspect.
    """

    def __init__(self, aspectname, message=None):
        self.aspectname = aspectname
        if message is None:
            message = ('Error when trying to search aspect named {}'
                       .format(repr(aspectname)))
        super().__init__(message)
```

It was completly fine at that time. But when I write the other patch an needs
to raise an `AspectLookupError`, I have an aspect object and want to write a
custom message, which I could do but a bit awkward because `aspectname` will be
filled just for placeholder. Maybe I should write both `aspectname` and
`message` as optional keyword arguments. But again, I am not really sure about
it.


### AspectList: Add `get()` method

<https://github.com/coala/coala/pull/4392>

Maybe the most complicated and long patch here.

The main goal is to make something like this possible:

```python
>>> aspects = aspectlist([Root.Smell.MethodSmell,
...                       Root.Smell.ClassSmell.DataClump()])

>>> aspects.get('MethodSmell')
<aspectclass 'Root.Smell.MethodSmell'>

>>> aspects.get('ClassSmell.DataClump')
<DataClump object(...) ...>

>>> aspects.get('BadSmell')
None
```

The idea is we can get a specific aspect (or its subaspect) from an AspectList.
This could really usefull for bears API so bear writer can check if any aspect
is exist (so they can turn on/off features) and to access taste of subaspect.

Here are example on how it could be used in bears.

```python
class SomeBear(LocalBear, aspects={
    'fix': Root.Metadata.CommitMessage
}):
    def run(self, file, filename, aspects:aspectlist=None):
        shortlog = aspects.get(Root.Metadata.CommitMessage.ShortLog)
        arg = ()
        if shortlog:
            arg += ('max_length=' + shortlog.max_length)
        # run bear with arg or something
```

This patch is separated into 2 commit


#### aspectbase: Add ``get()`` method]

To make things more nice, I don't directly create a `AspectList.get()` method in
AspectList. Instead, I create a more basic `aspectbase.get()` method that will
fetch a subaspect (whether it is a direct children or grandchildren) of a single
aspect. The idea is `AspectList.get()` will call `aspectbase.get()` for each
of the list item. The benefit is we can get detailed subaspect on a list AND
a single aspect without code duplication.

```python
class aspectbase:

    ...

    def get(self, subaspect):
        # Avoid circular import
        from .meta import isaspect, issubaspect

        # Accept `subaspect` as a string of aspect name
        if not isaspect(subaspect):
            # then convert it to aspect object
            subaspect = coalib.bearlib.aspects[subaspect]

        # Check if subaspect really exist under parent
        if not issubaspect(subaspect, self):
            # If not, just return None
            return None

        # It is pretty tricky, but we CANNOT search an subaspect instance
        # FROM another aspect instance.
        # For example:
        # >>> metadata = Root.Metadata('py') # An instance
        # >>> commitmessage = Root.Metadata.CommitMessage('py') # instance too
        # >>> metadata.get('commitmessage')  # normal, possible
        # >>> metadata.get(commitmessage)    # doesn't make sense, more on it later
        if isinstance(subaspect, aspectbase):
            raise AttributeError('Cannot search an aspect instance using '
                                'another aspect instance as argument.')

        # This was intentionally left as NotImplementedError because of
        # undesired behaviour when instancing an aspectclass that doesn't
        # instance the child aspect. So its better to return an error for now
        # rather than wrong result.
        if isinstance(self, aspectbase):
            raise NotImplementedError('Cannot access children of aspect '
                                     'instance.')

        # Get parent fully qualified name
        parent_qualname = (type(self).__qualname__ if isinstance(
                        parent, aspectbase) else self.__qualname__)
        # If the name of parent and child is same, then it was the same thing,
        # duh.
        if parent_qualname == subaspect.__qualname__:
            return parent

        # Trim common parent name, regex magic.
        aspect_path = re.sub(r'^%s\.' % parent_qualname, '',
                            subaspect.__qualname__)
        aspect_path = aspect_path.split('.')
        child = self
        # Traverse through children until we got our subaspect
        for path in aspect_path:
            child = child.subaspects[path]
        return child
```

There was a problem though. Ideally, `aspectbase.get()` should be callable from
BOTH of aspectclass (`Root.Metadata`) AND aspectclass instance
(`Root.Metadata('py')`). The above code is callable from aspect instance but NOT
from aspectclass because the `self` is not exist. After ~~discussion~~ help 
from @userzimmerman, I write a "descriptor" class for `aspectbase`.

Basically, with Python's descriptor we can create an object that act as another
class special type attribute with custom code that run when we access, modify,
or delete it. AFAIU, it was like custom setter getter in Java object that have
extra logic in it. The main difference is it was **tied to the class** itself,
so we can access it from instance and class, and the descriptor itself could
identify from where it was called. PERFECT!

More [source](http://intermediatepythonista.com/classes-and-objects-ii-descriptors)
[about it](https://www.smallsurething.com/python-descriptors-made-simple/).

So I modify code a ~~bit~~ lot.

```python
def get_subaspect(parent, subaspect):
    # Avoid circular import
    from .meta import isaspect, issubaspect
    if not isaspect(subaspect):
        subaspect = coalib.bearlib.aspects[subaspect]
    if not issubaspect(subaspect, parent):
        return None
    if isinstance(subaspect, aspectbase):
        raise AttributeError('Cannot search an aspect instance using '
                             'another aspect instance as argument.')
    if isinstance(parent, aspectbase):
        raise NotImplementedError('Cannot access children of aspect '
                                  'instance.')

    parent_qualname = (type(parent).__qualname__ if isinstance(
                       parent, aspectbase) else parent.__qualname__)
    if parent_qualname == subaspect.__qualname__:
        return parent

    # Trim common parent name
    aspect_path = re.sub(r'^%s\.' % parent_qualname, '',
                         subaspect.__qualname__)
    aspect_path = aspect_path.split('.')
    child = parent
    # Traverse through children until we got our subaspect
    for path in aspect_path:
        child = child.subaspects[path]
    return child

class SubaspectGetter:
    """
    Special "getter" class to implement ``get()`` method in aspectbase that
    could be accessed from the aspectclass or aspectclass instance.
    """

    def __get__(self, obj, owner):
        # obj is the aspectclass
        # owner is the aspectclass instance
        parent = obj if obj is not None else owner
        return functools.partial(get_subaspect, parent)

class aspectbase:

    get = SubaspectGetter()

    ...
```

I move the `aspectbase.get()` to module level function 
`get_subaspect(parent, subaspect)`. Then I Create the descriptor class
`SubaspectGetter` with `__get__` that call the `get_subaspect` function but
always use the aspectclass or instance (depend on if instance was exist on
caller) as the parent argument.

Then, in the `aspectbase` itself I define `SubaspectGetter` as class variable
named `get`.

Now its callable from both aspectclass and instance!!

```python
>>> import coalib.bearlib.aspects as coala_aspects

>>> coala_aspects['metadata'].get('commitmessage')
<aspectclass '...CommitMessage'>

>>> coala_aspects['metadata']('py').get('commitmessage')
NotImplementedError: ...
# should be <aspectclass '...CommitMessage'> but it was because of another bug.
```

And this commit was finished. Move to the next!


#### AspectList: Add ``get()`` method

This is a MUCH simpler commit because the dirty work was done by the last
commit.

I create a method in `AspectList`:

```python
def get(self, aspect):
    """
    Return first item that match or contain an aspect. See
    :meth:`coalib.bearlib.aspects.aspectbase.get` for further example.

    :param aspect: An aspectclass OR name of an aspect.
    :return:       An aspectclass OR aspectclass instance, depend on
                   AspectList content. Return None if no match found.
    """
    if not isaspect(aspect):
        aspect = coalib.bearlib.aspects[aspect]
    try:
        # iterate through list
        # call item.get() -> ``aspectbase.get``
        # return first item that is not None
        return next(filter(None, (item.get(aspect) for item in self)))
    except StopIteration:
        return None
```

And that was it.


### aspectModule: Add `get()` method

<https://github.com/coala/coala/pull/4412>

```python
def get(self, aspectname): 
    """ 
    Search and get an aspectclass from string of full or partial 
    qualified name of the aspect. Similiar to ``__getitem__``, but doesn't 
    raise exception for trying to search non existing aspect. 

    :param aspectname: 
        Name of the aspect that should be searched. 
    :raise MultipleAspectFoundError: 
        When multiple aspects with same name was found. 
    :return: 
        An aspectclass or None. 
    """ 
    try: 
        return self[aspectname] 
    except AspectNotFoundError: 
        return None 
```

In here, I just create a wrapper for `__getitem__` that doesn't raise error
when it not found the aspect. This was part of effort to make aspects API more
uniform (we have `get()` in aspectbase and AspectList too).


## Summary

So the groundwork for aspect API (more or less) is done. Next coding phase is
time to implement the aspect itself in coala processing part.
