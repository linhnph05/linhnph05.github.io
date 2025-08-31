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

# Payload
```python
print(__import__('os').popen('ls').read())
```
