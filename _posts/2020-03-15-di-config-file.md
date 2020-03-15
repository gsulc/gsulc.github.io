---
layout: post
title: "Dependency Injection through Configuration Files"
date: 2020-03-15 16:19:00 -0400
categories: programming design
---

#### With an example using YAML and Python

Chances are you've had this idea before: what if I can swap out parts of my software system based on a configuration file. Then, I could even add-in or take-away sub-systems. The software would be flexible, and technicians and other engineers can configure the system.

## Problem Statement
We wish to *deserialize* objects from a file. The classes may have constructors that take in objects of types defined in several libraries/modules, i.e. not just strings, numbers, bools, maps/dictionaries, and arrays. Some constructors objects that we want to re-use within the file. We want our file to be easily understood for technicians to edit.

## Solution
Sounds like a tall order! Luckily, the [PyYaml](https://pypi.org/project/PyYAML/) library has everything needed to do this. I imagine other YAML libraries are able to do so as well.
I like the elegance of Python. I will also use the YAML file format. [Previously, I wrote](config-files.html) about why YAML is a good choice and why XML is a bad choice for configuration. You should be comfortable with YAML, serialization, and Python before reading.

### Preparation
Install PyYAML with: `pip install pyyaml`

### Test Code
In this simple example, I split everything into three files: `classes.py` for class definitions, `config.yaml` for the configuration, and `program.py` for the main program.

First, a module of simple class definitions:

#### classes.py
```python
class A:
    def __init__(self, x):
        self.x = x
class B:
    def __init__(self, x, y):
        self.x = x
        self.y = y
```

Next, a YAML file:

#### config.yaml
```yaml
one:
  &anchor !!python/object:classes.A
  x: 1
two:
  !!python/object:classes.B
  x: 2
  y: *anchor
```
This is a bit more complex than a standard config file. The tag feature `!!` of YAML with the [pyyaml implementation](https://pyyaml.org/wiki/PyYAMLDocumentation) lets us deserialize into a `A` and `B` type *objects* defined in the *Python module*, "classes".

### Safely Loading
Here we safely load the program:

#### program.py
```python
import yaml
import classes

yaml.full_load('config.yaml')
print(type(objs['one'].x)) # <class 'int'>
print(type(objs['two'])) # <class 'classes.B'>
print((objs['one'] is objs['two'].y)) # True
print(f"{objs['one'].x} == {objs['two'].y.x}") # 1 == 1
print(yaml.dump(objs, default_flow_style=False))
```

Here is the output to show everything worked:
```
<class 'int'>
<class 'classes.B'>
True
1 == 1
one: &id001 !!python/object:classes.A
  x: 1
two: !!python/object:classes.B
  x: 2
  y: *id001
```

### Unsafely Loading
What if we have a lot of modules to load? Or modules are frequently swapped? I don't want to have a file that has 100+ lines of `import`s. What if a future engineer wants to add a new module of classes to use? They would have to add an `import` statement. How are they going to know to do that? You have to tell them or write documentation (which **will** grow out-of-date, regardless of what you tell yourself). If you are using a compiled language, you will have to recompile and redistribute new binaries. These problems can be avoided with `yaml.unsafe_load()`:

#### program.py
```python
import yaml
# no need to import your module

yaml.unsafe_load("config.yaml")
print(yaml.dump(objs, default_flow_style=False))
```

This is **unsafe** because arbitrary things can be loaded into memory. If the source of your YAML is not secure, do *not* do this. If you control the source and you don't expose it to the outside world (like allowing users to upload or override files over network connection), it is fine. It can be perfectly acceptable to pass on the security responsibilities to the consumer application. Convenience is often more valued than security.

### Friendlier YAML
It would be nice if we could make the YAML file look more like this:
```yaml
one:
  &anchor !A
  x: 1
two:
  !B
  x: 2
  y: *anchor
```

The easiest way to do this is to modify your classes to *inherit* from the `yaml.YAMLObject` class. But this creates a dependency on PyYaml and makes your classes strongly-coupled. It makes your code more brittle and confusing. Like so:

```python
import yaml # dependency

class A(yaml.YAMLObject): # strongly coupled
    yaml_tag = u'!A'
    def __init__(self, x):
        self.x = x
class B(yaml.YAMLObject): # strongly coupled
    yaml_tag = u'!B'
    def __init__(self, x, y):
        self.x = x
        self.y = y
```
As a designer, I would try to avoid this approach. If you know that you're never going to change your file format and don't need to adapt it, then maybe this is okay for you. The tradeoff is more appealing the less technical your users are likely to be. But I believe there is a way to do both based on the documentation. If I figure it out, I'll write about it.

## Other Solutions
You can use an *inversion of control* or *dependency injection* container to do this. But they're typically more complicated to use than `yaml.load_full(filename)`. The typical use-case is also a bit different: they usually seek to resolve dependencies strewn across an application. Whereas the use-case presented here only creates two objects for a single entity to use. If you wish, you can *make* this entity a dependency injection container.