---
layout: post
title: "Custom Tags in PyYaml"
date: 2020-04-05 15:41:00 -0400
categories: programming design
---

[Last post](di-config-file), I showed how to deserialize custom classes using PyYaml. But there was a lingering improvement that would be desireable in a real-world applications. It would be better if class decorations in YAML that look like `!!python/object:classes.A` could be replaced with declarations that look more like `!A` or even `!classes.A`, but without modifying the classes themselves.

PyYaml promotes adjusting the classes to achieve this, like so:
```python
# classes.py
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
But loose coupling allows an application to be more flexible and for the ability to use Python libraries without having to write wrapper classes around the library classes.

I worked out solutions for each style: with the module name and without. However, both methods require adding code to register every tag+class combination. I propose a way to remove that requirement later.

Here's the simple classes:
```python
# classes.py
class A:
    def __init__(self, x):
        self.x = x
class B:
    def __init__(self, x, y):
        self.x = x
        self.y = y
```

## With the Module Name
Let's start with what the YAML should look like.
```yaml
# config.yaml
one:
  &label !classes.A
  x: 1
two:
  !classes.B
  x: 2
  y: *label
```
To do this, I created some monkey-patches for yaml.

```python
# yamltagextensions.py
import yaml

def construct_py_object_from_tag(loader: yaml.Loader, node: yaml.Node):
    classname = node.tag.replace('!', '')
    return loader.construct_python_object(classname, node)

def add_py_constructor_to_tag(tag: str, loader: yaml.Loader = None):
    if loader is None:
        yaml.add_constructor(tag, construct_py_object_from_tag)
    else:
        loader.add_constructor(tag, construct_py_object_from_tag)

yaml.add_py_constructor_to_tag = add_py_constructor_to_tag
```

To use it:
```python
import yaml
import yamltagextensions

# Tags need to be registered first.
for tag in [u'!classes.A',  u'!classes.B']:
    yaml.add_py_constructor_to_tag(tag)

objs = yaml.unsafe_load(document)

print((objs["one"] is objs["two"].y)) # True
print(type(objs["one"].x)) # <class 'int'>
print(type(objs["two"])) # <class 'classes.B'>
print(f"{objs['one'].x} == {objs['two'].y.x}") # 1 == 1
print(yaml.dump(objs, default_flow_style=False))
```

## Without the Module Name
YAML
```yaml
# config.yaml
one:
  &label !A
  x: 1
two:
  !B
  x: 2
  y: *label
```

```python
# yamltagextensions.py
import yaml

_classes = dict()

def construct_py_object_from_class(loader: yaml.Loader, node: yaml.Node):
    global _classes
    classname = node.tag.replace('!', '')
    return loader.construct_python_object(_classes[classname], node)

def add_py_constructor_for_class(class_, loader: yaml.Loader = None):
    global _classes
    name = class_.__name__
    _classes[name] = class_.__module__ + '.' + name

    if loader is None:
        yaml.add_constructor('!' + name, construct_py_object_from_class)
    else:
        loader.add_constructor('!' + name, construct_py_object_from_class)

yaml.add_py_constructor_for_class = add_py_constructor_for_class
```

This solution uses a global dictionary to map a class name to a class name + module required by `yaml.Loader.construct_python_object()`. This is not very nice. A real solution will require extra checks.

```python
import yaml
import yamltagextensions

# Classes need to be registered first.
for class_ in [classes.A, classes.B]:
    yaml.add_py_constructor_for_class(class_)

objs = yaml.unsafe_load(document)

print((objs["one"] is objs["two"].y)) # True
print(type(objs["one"].x)) # <class 'int'>
print(type(objs["two"])) # <class 'classes.B'>
print(f"{objs['one'].x} == {objs['two'].y.x}") # 1 == 1
print(yaml.dump(objs, default_flow_style=False))
```

The other issue with this is that `unsafe_load()` is used, but we're specifying what types can get be loaded. If the modules were loaded, this would be safe.

## Without Registration
Of course, it would be most ideal to avoid having to register any tags at all. This requires modifying the PyYaml library. The easiest method I found to override is `yaml.constructors.SafeConstructor.construct_undefined`. I modified it directly and it worked:

```python
def construct_undefined(self: SafeConstructor, node):
    try:
        classname = node.tag.replace('!', '')
        return self.construct_python_object(classname, node)
    except:
        raise ConstructorError(None, None,
            "could not determine a constructor for the tag %r" % node.tag,
            node.start_mark)
```

For some reason, when I tried monkey-patching, it did not work.
```python
from yaml.constructor import ConstructorError, SafeConstructor

def construct_undefined(self: SafeConstructor, node):
    try:
        classname = node.tag.replace('!', '')
        return self.construct_python_object(classname, node)
    except:
        raise ConstructorError(None, None,
            "could not determine a constructor for the tag %r" % node.tag,
            node.start_mark)

SafeConstructor.construct_undefined = construct_undefined
```

I'd love for someone to tell me what is wrong.

One thing to note is: you are overriding the `SafeConstructor`, making it unsafe. So keep that in mind.