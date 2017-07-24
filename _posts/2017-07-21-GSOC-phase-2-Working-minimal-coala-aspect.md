---
title: "GSOC Phase 2 - Working MVP Coala Aspect"
layout: single
excerpt: "In phase 2, my target is to make aspect-based configuration *works*. 
Maybe not with its full power and capabilities, but at least the core 
foundation must ready in coala codebase and prove that aspect could run."
sitemap: true
author_profile: false
category: "developer"
tags:
  - "English"
  - "coala"
---

This post is part of my GSOC journey in coala.

In phase 2, my target is to make aspect-based configuration *works*. Maybe not
with its full power and capabilities, but at least the core foundation must
ready in coala codebase and prove that aspect could run.

Starting from my previous phase, I have 3 crucial milestone to reach, which
is:

- Intepret aspect-based configuration
- Passing aspects into bear
- Choosing bear based on aspects

Even though the todo is short, I meet with miscellaneous supporting works, 
few surprising but interesting "bugs", and design decisions. And for concrete
proof, I make a repository that contain an example project that could be run
with `.coafile` with aspect based configuration. You can see it in
[adhikasp/demo-aspect](https://github.com/adhikasp/demo-aspect/).

Ok then... let's move to the code.


## Intepret aspect-based configuration

For coala to collect the right bears and pass meaningfull data into each bears,
it need to correctly intepret aspect related configuration. Aspect configuration
is special (and also give some headache too) because it is written in
`.coafile`, like other setting, but they must be preprocessed first before it is
ready to be used/read.


### aspectbase: Recursively instance aspect children

<https://github.com/coala/coala/pull/4427>

First thing to do is to fix unintended behaviour in aspect instantiation.
The old behaviour is when we instance some leaf aspects, the child is not
instanced. This is problematic because in bears, we want to access leaf aspect
and its taste but `.coafile` configuration could define a arbitrary node
aspects. The result is we can't access leaf aspects that have custom taste
because it was never instanced in the first place and the information from
`.coafile` doesn't have place to be stored.

The solution is to recursively instance aspect and all of its children. Also,
we pass the `**taste_values` too so a single aspect instantiation could set
taste for all of its hierarchy.

Here is the code

```python
class aspectbase:

  def __init__(self, language, **taste_values):
    ...

    # Recursively instance its subaspects too
    instanced_child = {}
    for name, child in self.subaspects.items():
      instanced_child[name] = child(language, **taste_values)
    self.__dict__['subaspects'] = instanced_child
```

Pretty straight forward. Every aspects will read `**taste_value` dict and just
take whatever value they need, so there is no harm to pass it down the tree.
This make declaring aspects and then set a custom taste for one of its children
aspects in `.coafile` became possible. We overwrite the `.subaspects` with dict
full of instanced child so we can access them later.

There is a (positive) side effect of this change. The `get_subaspect()`
function which I wrote in
[my previous PR](https://github.com/coala/coala/pull/4412) could behave
correctly :D


### AspectList: Add `exclude` attribute

<https://github.com/coala/coala/pull/4439>

Next step is extending AspectList have a `exclude` property that hold a list
of aspects that was "excluded" from the AspectList itself. The main idea is
to make users could make exception on what aspects they want to run in
`.coafile`.

```python
class AspectList(list):

  def __init__(self, seq=(), exclude=None):
    super().__init__((item if isaspect(item) else
                      coalib.bearlib.aspects[item] for item in seq))

    self.exclude = AspectList(exclude) if exclude is not None else []

  def __contains__(self, aspect):
    for item in self:
      if issubaspect(aspect, item):
        # Make sure it's not excluded
        return aspect not in self.exclude
    return False    
    
  def get(self, aspect):
    ...
    # Make sure it's not excluded
    if aspect in self.exclude:
      return None
```

Again, not a complicated code, but this one is pretty tricky because it change
how other function must behave, which is `__contains__()` and `get()` function.
We must make sure treat anything that was listed in `exclude` like it was not
contained in AspectList at all.

With `exclude`, we provide a nice API for coala to store list of excluded
aspects in configuration.


### Extract aspects from configuration

<https://github.com/coala/coala/pull/4397/commits/1ce4dd96a30c2acb913fb9a9f05cfc7dcac87c0f>

Actually this is part of next section PR but with how I write this article it
make more sense to write it in this section :/

So in this phase coala's aspect have complete API to store aspects into
AspectList. Now its time to extract the information from `.coafile` into the
AspectList itself.

```python
def extract_aspects_from_section(section):
  """
  Extracts aspects and their related settings from a section and create an
  AspectList from it.
  :param section: Section object.
  :return:        AspectList containing aspectclass instance with
                  user-defined tastes.
  """
  # Get section data from configuration
  aspects = section.get('aspects')
  language = section.get('language')

  # Skip aspects initialization if not configured in section
  if not len(aspects):
    return None

  if not len(language):
    raise AttributeError('Language was not found in configuration file. '
                          'Usage of aspect-based configuration must '
                          'include language information.')

  aspect_instances = AspectList(exclude=section.get('excludes'))

  for aspect in AspectList(aspects):
    # Search all related tastes in section.
    tastes = {name.split('.')[-1]: value
              for name, value in section.contents.items()
              if name.lower().startswith(aspect.__name__.lower())}
    aspect_instances.append(aspect(language, **tastes))

  return aspect_instances
```

With `extract_aspects_from_sections()` function, we can get data we need
to choose the right bears and make aspectized bear do their work correctly :+1:


## Passing aspects into bear

### Passing the data

<https://github.com/coala/coala/pull/4397/commits/1ce4dd96a30c2acb913fb9a9f05cfc7dcac87c0f>

Still from the same commit as before. We want to pass all of those precious data
extracted from section so it can be used by bears.

It was pretty complicated.

My first attempt is to call `extract_aspects_from_sections()` in
gather_configuration phase, and then change those function return value and
other chained function until it was passed into function who responsible to
instance and execute the bear.

Aaaand it was bad because it became to complicated and breaking API, which is
*very bad* because it will make coala-quickstart broke. The main cause of API
breaking is because I change some key function return format, which mainly use
tuple. So a function who return 2 tuples will return 3 now.

After discussion with my mentors, we decide to embed the aspects information
into `Section` class as a new instance attribute. This is a very simple
approach and still have room for further improvement if needed. 

```python
def aspectize_sections(sections):
  """
  Search for aspects related setting in a section, initialize it, and then
  embed the aspects information as AspectList object into the section itself.

  :param sections:  List of section that potentially contain aspects setting.
  :return:          The new sections.
  """
  for _, section in sections.items():
    section.aspects = extract_aspects_from_section(section)
  return sections
```

I was pretty amazed. What was originally a ~30 lines patch in many different
functions and files became 11 lines function, which is a good news. 

Now bears can access the aspects like this

`.coafile`
```
[all]
files = **.py
bears = coalaBear
aspects = coalaCorrect, shortlog.colonexistence
language = Python 3.6
shortlog.colonexistence.shortlog_colon = false
```

```python
class SomeBear(LocalBear):
  def run(self, filename, file):
    print(self.section.aspects)
    # [<...coalaCorrect object at 0x...>, <...ColonExistence
    #  object at 0x...>]
    print(self.section.aspects[1])
    # <ColonExistence object(shortlog_colon=False) at 0x...>
    print(self.section.aspects[1].shortlog_colon)
    # True
```

Overall the `.coafile` syntax is not nearly accurate to cEP5 but at least we
got something going on :D


### Weird bug 1: `Language` is not pickleable!

So I write above code then run the test suite, hope everything is green and
well. But of course, there is hard rule for programming that everything
never work on first try. I got some cryptic error code, `PickleError`.

After much investigation I found out that python is complaining that I try to
give them some object that was not pickleable. Coala run with multithreading
support and apparently they "pickle" the setting/Section object so it can be
accessed by different bears that running parallel in different threads.

My code add `aspect` and `Language` object into into `Section`. Apparently,
`Language` object is not pickleable. My rough idea is because its use `Version`
package which use `Infinity` class that was not pickleable.

I tried to work around this. First thing is to try to make the data of
`Language` pickleable. But I realize that this will involve big changes and
effort. And also I was unfamiliar with `Language` module. Luckily, I found
second approach, which is to declare `__reduce__()` function that could
reconstruct a `Language` object from function call and list of needed
parameters (only a string of language name, very simple!).

```python
def __reduce__(self):
  return (Language.__getitem__, (str(self),))
```

And then the tests is passing again.


## Choosing bear based on aspects

<https://github.com/coala/coala/pull/4532>

Bear can have aspects. But we also need the bear to be picked by aspects too!

This is a pretty big PR that have many commits in it.


### meta: Add Languages to bears and AspectList: Connect with holder bear

<https://github.com/coala/coala/pull/4490>

This 2 commit is actually a PR I take over from @pratyuprakash, so credits for
him too :D

In this commit we expand `Bear`'s metaclass to have `language` attribute which
hold `Language` object that tell what language is this bear could analyze.


### Test: Add AspectTestBear

Now that we have many aspects function in core, we need to test those with an
aspect ready bear!

```python 
class AspectTestBear(LocalBear, aspects={
    'detect': [
        Root.Redundancy.UnusedVariable.UnusedGlobalVariable,
        Root.Redundancy.UnusedVariable.UnusedLocalVariable,
    ]
}, languages=['Python']):
    LANGUAGES = {'Python'}
    LICENSE = 'AGPL-3.0'

    def run(self, filename, file, config: str=''):
        """
        Bear that have aspect.
        :param config: An optional dummy config file.
        """
        yield Result.from_values(
            origin=self,
            message='This is just a dummy result',
            severity=RESULT_SEVERITY.INFO,
            file=filename,
            aspect=Root.Redundancy.UnusedVariable.UnusedLocalVariable)
```


### Bear.py: Make `languages` JSON compliant

So `Language` is a bit of difficult class. when I run the test again, few of
JSON related test is failing. I was confused at first. I don't even change
them!!

Long story short, it was caused by `Language` object that was embedded in bear
(`AspectTestBear` from before) can't be converted to JSON string. Wow.

Conveniently, the `Bear` class already have `__json__` function that override
its behaviour when converted to JSON. So I just add 2 simple line to tell it
how to convert a `Language` object too.

```python
if hasattr(cls, 'languages'): 
  _dict['languages'] = (str(language) for language in cls.languages)
```


### aspects: Create `get_leaf_aspects` method


Create a function that could "implode" (or is "explode" the correct term?) an
arbitrary aspects into list of its leaf aspect. The main idea here is to make
the bear <-> aspect matching more simple. Just iterate through the leaf aspect,
check if any bear support it, then cross it from the list! No fancy algorithm
or overlapping aspects.


### Collectors: Create basic bear collector by aspect

## Aspect Demo

Now I will explain about the demo.

Again, the link is <https://github.com/adhikasp/demo-aspect/>

It is a minimal project that have few files:

- `demo-aspect.py` - the faulty code
- `.coafile` - the configuration for coala
- `.travis.yml` - CI script that run coala and make sure it works

Here is the content of `demo-aspect.py`

```python
# Unused import!! Remove cos
from math import sin, cos
# not removing this because we can set ``remove_only_standard_package`` taste!
import UnusedButNonStandardPackage

def foo():
    # Unused and can be removed.
    # BUT, not removed because we exlude UnusedLocalVariable
    x = 1
    return sin(5)
```

Aaand the config `.coafile`

```
[python]
files = *.py
aspects = Redundancy
language = Python
excludes = UnusedLocalVariable
Redundancy.remove_only_standard_package = True
```

There is no bears listed in `.coafile` but if we run it, coala will pick
(aspectized) PyUnusedCodeBear with our defined setting through taste!

