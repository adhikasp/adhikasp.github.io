---
title: "Designing configuration for coala's aspect"
layout: single
excerpt: "With the emerge of aspect project in coala, come the need to redefine how
configuration works and define various new parameter that should control how
the aspect should run."
sitemap: true
author_profile: false
category: "developer"
tags:
  - "English"
  - "coala"
---

This post is part of my GSOC journey in coala.

With the emerge of aspect project in coala, come the need to redefine how
configuration works and define various new parameter that should control how
the aspect should run.

Currently, the mockup for those configuration could be found on [cEP-0005](https://github.com/coala/cEPs/blob/53720cc8d792bb73206e33977ee7e1f2fb1414a9/cEP-0005.md#mockup)
and look like this

```python
[all]
# Values can be added to inherited ones only, not within the section because
# a configuration should describe a state and not involve operations.
# A system wide coafile can define venv, .git, ... and we would recommend to
# always use += for ignore even in the default section to inherit those values.
# += always concatenates with a comma.
ignore += ./vendor

max_line_length = 80

# This inherits all settings from the `all` section. There is no implicit
# inheritance.
[all.python]
language = Python
files = **.py
aspects = smell, redundancy.clone  # Inclusive

[all.c]
language = C
files = **.(c|h)
ignore_aspects = redundancy  # Exclusive
```

This mockup provide basic example on how aspect configuration could work. Its 
key feature is:

- New parameter `aspect` and `ignore_aspects` to define list of aspect that
  coala should analyze.
- New parameter `language` to help coala choose the right bear that could 
  analyze a specific language.
- A taste could defined by writing its name directly (`max_line_length = 80`).
- Specifying `bears` manually is still possible.
- Explicit section inheritance (`[parent.child]`).

Again, this configuration is still a mockup. I think, there are still a few way
to further enhance this configuration.


## Default behaviour

> One important aspect of coala and its usability is that the configuration
> of a new language is as easy as possible.

IMO, usability is a very crucial factor on making coala more widely used in 
open source project. People like to explore new tools, but if configuring these
tools take too much time or too complex, they tend to pass it and get the next 
tools on the list. coala must shallow its learning curve.

In that regard, I want to make the configuration powerfull and compact, but 
still provide option to tweak it in detail. It's should be possible for people
to run coala with the most minimal line of configuration, or even with no 
explicit configuration nor initial setup beforehand.

For coala to analyze aspect properly, we need to know:

1. What aspect is choosen
2. Which files should analyzed
3. What languages it is written in, and
4. (optional) Defining aspect's taste, how a correct aspect should look like

This could be detailed further...


### Defining aspect

An aspect could be written by its fully qualified name (`Root.Some.MyAspect`) or
partially qualified name (`MyAspect` or `Some.MyAspect`). Defining a parent node
mean all of its children node will be included too.

Defining taste could be done with writing the taste name as the parameter name
and assign a value to it. In case of multiple aspects have the same taste name, 
ambiguity must be resolved by prefixing the taste name with its aspect name. 
For example, `max_length` is a taste of `Shortlog` and `LineLength`. Thus the 
`Shortlog` taste would be defined like `Shortlog.max_length = 50`.


### Defining language

While defining language, we could use coalang (could be found in module 
`coalib.bearlib.languages.definitions`) to provide flexibility on writting the
language name. For example, `C++` could also written by `CPP`, `CXX`, or even
`CPlusPlus` and they will still refer to the same progamming language.

Other advantadge is in the case of language that have multiple version (like
python2 vs python3), coalang could provide a default version number or users
could define it themself with keyword `version`. Note that defining version for
language is not mandatory. Its neccessary in case of project that should be 
analyzed under specific version, like python3 project that should analyzed
by python3 only bear.


### Defining files

IMO explicitly defining file in section is not always necessary because we can
get list of file extension from coalang. By defining `language = CPP`, then it 
also mean implicitly `files = **.c, **.cpp, **.h, **.hpp`. Of course this could
be overwritten by explicitly define `files` parameter in configuration.


## Backward compatibility for choosing bear by its name

There is some unresolved problem on defining bear. The problem is gathering the
option configuration for those bear. In aspect based configuration, bear option
will collected through taste, but in old configuration, option is given as a 
kind of "global" parameter in section. It will be a enormous task to write a
mapping function from old parameter and its taste pair.

For the time being, I think the most easy approach is to make conditionally 
collection on each bear with something like this:

```python
class LineLengthBear(LocalBear, aspects=[
  'detect': LineLength
]):
    def run(self, filename, file, aspects, min_length):
        if LineLength in aspects:
            # Overwrite with the aspect's taste if it exist
            min_length = aspects.get(LineLength).min_length
        
        # do something with min_length
```


## Conclusion

To write a aspect configuration, we at least need to know **what aspect** we 
want to analyze and **what language** it's written in. We can configure it 
further by defining list of files and list of option (aspect's taste) for each
aspect.

This is my first time trying to write a configuration format and it will be far
from perferct or have some unforseen edge case. Hopefully, all the bugs and 
weird edge cases could be tackled in the future while the the coding is in 
progress.
