---
title: Learning about server side template injection in python
published: 2025-09-02
description: 'Deep dive into python objects'
image: ''
tags: [jinja, python, ssti]
category: 'python'
draft: true
lang: ''
---

# Magic method
## __new__(cls, *args, **kwargs)
- A static method (though usually defined as a class method) responsible for creating and returning a new instance of the class.
- It is called before __init__.
- Usually returns an instance of cls (or a subclass).

## __init__(self, *args, **kwargs)
- Called after the object is created.
- Responsible for initializing attributes of the object.
- It does not return anything (must return None).

# Payload
```python
print(__import__('os').popen('ls').read())
```
