---
title: "GSOC Phase 1 Mid - Automatic star import expanding on autoflake"
layout: single
excerpt: "In this week, I was [writting a patch](https://github.com/myint/autoflake/pull/18) 
into [autoflake](https://github.com/myint/autoflake) for a new feature that
could expand a wildcard import to specify each name that really was used in the
code."
sitemap: true
author_profile: false
category: "developer"
tags:
  - "English"
  - "coala"
---

This post is part of my GSOC journey in coala.

In this week, I was [writting a patch](https://github.com/myint/autoflake/pull/18) 
into [autoflake](https://github.com/myint/autoflake) for a new feature that
could expand a wildcard import to specify each name that really was used in the
code.

## Autoflake and how it work

Short introduction. Autoflake is a tool for automatically remove unused imports,
unused variables, and useless pass statemant from Python code. It was mainly
dependent on [pyflakes](http://pypi.python.org/pypi/pyflakes), a python code
checker, which provide the analysis foundation to check for the faulty code.
Autoflake itself just handle the code manipulation part based on pyflakes
report and actually remove those.

For example, a code like this

```python
import math
import re
math.sin(1)
```

The `import re` statement will be removed because it was detected as unused.

Here are a simplified version on the step autoflake take when examining a code:

1. Send the file content to be analyzed by pyflakes via its API.
2. Read the result and filter it according to message type. Pyflakes can detect
   many type of error, but autoflake only care about few specific error which
   is `pyflakes.messages.UnusedImport` and `pyflakes.messages.UnusedVariable`.
   We save the line number of each error finding for further usage.
3. Iterate the lines of file. For each line, we check if the line have any 
   error that was found in step 2 before.
4. Each type of error (marked by its corresponding pyflakes message type in
   step 2) will be handled by different function that could fix it. Regex is
   used for the string manipulation. The function will return a modified line,
   empty line, or a `pass` statement.
5. Repeat step 1-4 until pyflakes doesn't find error. This iteration process
   is used because autoflake is not directly "remove" a variable or import,
   instead, it will swap it with `pass` statement to avoid a case like empty
   block (`if`, `class` declaration, etc). The next iteration then will check
   if some of the useless `pass` statement could be safely removed.

Automatic code manipulation like autoflake is a sensitive thing and a mistake
on its algorithm could result make a fine, albeit dirty code into a completly
broken code. Which is why autoflake have few different test kit to make sure
it was working correctly.

- [`test_autoflake.py`](https://github.com/myint/autoflake/blob/master/test_autoflake.py) -
  unit test that check each function in autoflake. This make sure that every
  function is working correctly.
- [`test_fuzz.py`](https://github.com/myint/autoflake/blob/master/test_fuzz.py) -
  run autoflake against Python code that it find in system path (like loads of
  Python standard library) and make sure that it doesn't broke any syntax nor
  introduce more pyflakes error.
- [`test_fuzz_pypi.py`](https://github.com/myint/autoflake/blob/master/test_fuzz_pypi.py) -
  Same like `test_fuzz.py` above, but instead of searching Python code in system
  path, it download a the most recently updated PyPi packages and try to fix it.

`test_fuzz.py` and `test_fuzz_pypi.py` both really important to make sure
autoflake doesn't broke a real world Python program.


## Stop about autoflake, what do **I** do in the patch?

Here are the [issue](https://github.com/myint/autoflake/issues/14) that propose
the new feature.

![issue screenshot](https://i.imgur.com/9AJoEdD.png)

First, let us understand what is this feature about.

Essentially, autoflake should define implicitly list of name that was imported
and used from a wildcard (`*`) import. A wildcard import is something like
`from <module> import *`. Usage of wildcard import should be avoided according
to [PEP8](http://legacy.python.org/dev/peps/pep-0008) and general sentiment
of my [quick](https://stackoverflow.com/questions/3615125/should-wildcard-import-be-avoided)
[Google](https://stackoverflow.com/questions/22313924/why-how-does-named-vs-wildcard-import-affect-parameters)
[search](http://python.net/~goodger/projects/pycon/2007/idiomatic/handout.html#importing),
except in a few notable cases.

So, how exactly this feature will work? I think an example will make it easier
to understand.

A code like this...

```python
from math import *
sin(0)
cos(0)
```

will have its wildcard import expanded to explicitly say what exactly list of
names that was imported from **other** module and is **used** in this code. In
this case, we use `cos` and `sin` function from `math`. We do not use any other
function from `math` module and doesn't want it to dirty our root namespace.
Then it should became like this...

```python
from math import cos, sin
sin(0)
cos(0)
```

But, how could we know what names is undefined? If we run that code into
pyflakes, we will get the following report:

```
test.py:1: 'from math import *' used; unable to detect undefined names
test.py:2: 'sin' may be undefined, or defined from star imports: math
test.py:3: 'cos' may be undefined, or defined from star imports: math
```

As we see, pyflakes could point out which names was undefined and the possible
origin of those.

However, there will be some limitation. Because of pyflake only parse the code
without actually run/intepret it, they don't really know from what moudle a name
could possibly came if there exist more than single star import.

For example

```python
from math import *
from re import *
sin(0)
cos(0)
```

Pyflakes will report

```
test.py:1: 'from math import *' used; unable to detect undefined names
test.py:2: 'from re import *' used; unable to detect undefined names
test.py:3: 'sin' may be undefined, or defined from star imports: math, re
test.py:4: 'cos' may be undefined, or defined from star imports: math, re
```

Now, the undefined name could possibly came from `math` **or** `re`. We don't
exactly know which one and thus we can't automatically expand those without
risk of choose the wrong one. The feature limitation is the expanding only
could be done when there was exactly one star import on the whole file.


## The patch code itself

### Filtering the pyflake report message

As the first step, we need to make autoflake know about the new error type that
relate to star import usage. This could be done by declaring new function
that filter list of pyflakes message object to choose our specific error type.

```python
def star_import_used_line_numbers(messages): 
    """Yield line number of star import usage""" 
    for message in messages: 
        if isinstance(message, pyflakes.messages.ImportStarUsed): 
            yield message.lineno 
 
def star_import_usage_undefined_name(messages): 
    """Yield line number, undefined name, and its possible origin module""" 
    for message in messages: 
        if isinstance(message, pyflakes.messages.ImportStarUsage): 
            undefined_name = message.message_args[0] 
            module_name = message.message_args[1] 
            yield (message.lineno, undefined_name, module_name) 
```

I write 2 filtering function for this. `star_import_used_line_numbers()` will
find the line number where the import star syntax `from <module> import *` was
used. `star_import_usage_undefined_name()` in the other hand will find the
line number of undefined name that probably came from star import. It also
outputing additional info like the name itself and the possible module origin
which both was passed by pyflakes.

The content of the function itself was pretty standard and I think was self-
explanatory.


### Filtering the errors and group up undefined names 

Next step is call the filtering function above to filter the pyflakes report.
This was done `filter_code()` function, which is one of the most crucial
function in autoflake.

```python
# On/off switch for the feature.
if expand_star_import: 
    # Search the star import usage in code
    marked_star_import_line_numbers = frozenset(
        star_import_used_line_numbers(messages)) 
    if len(marked_star_import_line_numbers) > 1:
        # Auto expanding only possible for single star import.
        # If we found more than 1, then just clear the finding and don't
        # try to fix it.
        marked_star_import_line_numbers = frozenset()
    # Actually this `else` block could be omitted.
    else:
        # List to hold undefined name that should be written in
        # import statement
        undefined_names = []
        # Iterate through the undefined name that we find in code...
        for line_number, undefined_name, _ \
                in star_import_usage_undefined_name(messages):
            # and then append it to our list.
            undefined_names.append(undefined_name)
        # If we doesn't found any undefined name then don't try to do anything.
        if not undefined_names:
            marked_star_import_line_numbers = frozenset()
# Else block for off-ing this feature
else: 
    marked_star_import_line_numbers = frozenset()
```

Then autoflake will iterate each line in code to try fix problem in each of it.

```python
for line_number, line in enumerate(sio.readlines(), start=1):
    if '#' in line:
        yield line
    elif line_number in marked_import_line_numbers:
        yield filter_unused_import(
            line,
            unused_module=marked_unused_module[line_number],
            remove_all_unused_imports=remove_all_unused_imports,
            imports=imports,
            previous_line=previous_line)
    elif line_number in marked_variable_line_numbers:
        yield filter_unused_variable(line)
    # Here is my addition
    # Basically if the corresponding line is detected to have a star import
    # then...
    elif line_number in marked_star_import_line_numbers:
        # FIX IT
        yield filter_star_import(line, undefined_names)
    else:
        yield line
```

### Fix the problem

The core part that actually do the work. This function will take a line
containing the problematic star import and replace the wildcard with list of
name that was otherwise undefined in the code.

```python
def filter_star_import(line, marked_star_import_undefined_name):
    # Remove duplicate and sort it for a nice alphabetical order.
    undefined_name = sorted(set(marked_star_import_undefined_name))
    # Regex to replace * with list of undefined name, separated by comma. 
    return re.sub(r'\*', ', '.join(undefined_name), line) 
```

### Add command line argument and others

In this part, I modify the command line argument parser to accept new option to
set the feature as on/off.

```python
parser.add_argument('--expand-star-import', action='store_true', 
    help='expand wildcard star import with undefined names') 
```

And then I modify a few function signature to make sure it was correctly
passed into the `filter_code()` function.

### Testing

Last but not least, I write additional unit test for my new function. Then I run
the fuzzy test to make sure it work correctly in some real Python code.

Unfortunately, there was a bug for some special case. It is not from my code
nor implementation, but it caused by upstream bug from pyflakes which 
incorrectly provide a wrong report and make the fix function became broke. But
after consulting with my mentor @myint, it was considered too specific case and
I could ignore it.

The bug itself was documented in 
[my pull request](https://github.com/myint/autoflake/pull/18) if you want to
read about it.


## Conclusion

It was my ~~first~~ second time reading and writing code for a automatic code 
linter program, and it was a pretty interesting experience for me. My first time
is on autoflake too, when I submitted a patch for enhancment on removing name
in `from <module> import <few name>`. I learn alot from this. Parsing a code is
tricky because we must think of every possible way that programmer could write
the syntax and many of edge cases. For now, I just learn parsing with regex and
it was already pretty complicated.

Overall, I was interested about linter in general. It's pretty cool that we can
write a code for repairing our other code.
