---
title: Python cheat sheet
published: 2025-09-05
description: ''
image: ''
tags: [python, cheatsheet]
category: 'python'
draft: true
lang: ''
---

# Main
```python
ord("A") = 65
chr(65) = 'A'
hex(100) = '0x64'
hex(100)[2:] = '64'
" ".join(['a', 'b']) = "a b"
dir(str) = List of all the available methods
'abc’[-1] = ‘c’
'abc’[1:3] = ‘bc’ from [1] to [2]
"qwertyuiop"[:-1] = 'qwertyuio'
```

# Tuples
```python
t1 = (1,'2,'three')
t2 = (5,6)
t3 = t1 + t2 = (1, '2', 'three', 5, 6)
(4,) = Singelton
d = () empty tuple
d += (4,) --> Adding into a tuple
CANT! --> t1[1] == 'New value'
list(t2) = [5,6] --> From tuple to list
```

# List (array)
```python
d = [] empty
a = [1,2,3]
b = [4,5]
a + b = [1,2,3,4,5]
b.append(6) = [4,5,6]
tuple(a) = (1,2,3) --> From list to tuple
```

# Dictionary
```python
d = {} empty
monthNumbers={1:’Jan’, 2: ‘feb’,’feb’:2}—> monthNumbers ->{1:’Jan’, 2: ‘feb’,’feb’:2}
monthNumbers[1] = ‘Jan’
monthNumbers[‘feb’] = 2
list(monthNumbers) = [1,2,’feb’]
monthNumbers.values() = [‘Jan’,’feb’,2]
keys = [k for k in monthNumbers]
a={'9':9}
monthNumbers.update(a) = {'9':9, 1:’Jan’, 2: ‘feb’,’feb’:2}
mN = monthNumbers.copy() #Independent copy
monthNumbers.get('key',0) #Check if key exists, Return value of monthNumbers["key"] or 0 if it does not exists
```

# map, zip, filter, lambda, sorted and one-liners

**Map** is like: \[f(x) for x in iterable] --> map(tutple,\[a,b]) = \[(1,2,3),(4,5)]\
m = map(lambda x: x % 3 == 0, \[1, 2, 3, 4, 5, 6, 7, 8, 9]) --> \[False, False, True, False, False, True, False, False, True]

**zip** stops when the shorter of foo or bar stops:

```
for f, b in zip(foo, bar):
    print(f, b)
```

**Lambda** is used to define a function\
(lambda x,y: x+y)(5,3) = 8 --> Use lambda as simple **function**\
**sorted**(range(-5,6), key=lambda x: x\*\* 2) = \[0, -1, 1, -2, 2, -3, 3, -4, 4, -5, 5] --> Use lambda to sort a list\
m = **filter**(lambda x: x % 3 == 0, \[1, 2, 3, 4, 5, 6, 7, 8, 9]) = \[3, 6, 9] --> Use lambda to filter\
**reduce** (lambda x,y: x\*y, \[1,2,3,4]) = 24

```
def make_adder(n):
	return lambda x: x+n
plus3 = make_adder(3)
plus3(4) = 7 # 3 + 4 = 7

class Car:
	crash = lambda self: print('Boom!')
my_car = Car(); my_car.crash() = 'Boom!'
```

mult1 = \[x for x in \[1, 2, 3, 4, 5, 6, 7, 8, 9] if x%3 == 0 ]


# Assert()
If the condition is false the string will be printed in the screen

```python
def avg(grades, weights):
    assert not len(grades) == 0, 'no grades data'
    assert len(grades) == 'wrong number grades'
```

# Generators, yield
A generator, instead of returning something, it "yields" something. When you access it, it will "return" the first value generated, then, you can access it again and it will return the next value generated. So, all the values are not generated at the same time and a lot of memory could be saved using this instead of a list with all the values.

```python
def myGen(n):
    yield n
    yield n + 1
g = myGen(6) --> 6
next(g) --> 7
next(g) --> Error
```

# Regular Expresions

import re\
re.search("\w","hola").group() = "h"\
re.findall("\w","hola") = \['h', 'o', 'l', 'a']\
re.findall("\w+(la)","hola caracola") = \['la', 'la']

**Special meanings:**\
. --> Everything\
\w --> \[a-zA-Z0-9\_]\
\d --> Number\
\s --> WhiteSpace char\[ \n\r\t\f]\
\S --> Non-whitespace char\
^ --> Starts with\
$ --> Ends with\
\+ --> One or more\
\* --> 0 or more\
? --> 0 or 1 occurrences

**Options:**\
re.search(pat,str,re.IGNORECASE)\
IGNORECASE\
DOTALL --> Allow dot to match newline\
MULTILINE --> Allow ^ and $ to match in different lines

re.findall("<.\*>", "\<b>foo\</b>and\<i>so on\</i>") = \['\<b>foo\</b>and\<i>so on\</i>']\
re.findall("<.\*?>", "\<b>foo\</b>and\<i>so on\</i>") = \['\<b>', '\</b>', '\<i>', '\</i>']

IterTools\
**product**\
from **itertools** import product --> Generates combinations between 1 or more lists, perhaps repeating values, cartesian product (distributive property)\
print list(**product**(\[1,2,3],\[3,4])) = \[(1, 3), (1, 4), (2, 3), (2, 4), (3, 3), (3, 4)]\
print list(**product**(\[1,2,3],repeat = 2)) = \[(1, 1), (1, 2), (1, 3), (2, 1), (2, 2), (2, 3), (3, 1), (3, 2), (3, 3)]

**permutations**\
from **itertools** import **permutations** --> Generates combinations of all characters in every position\
print list(permutations(\['1','2','3'])) = \[('1', '2', '3'), ('1', '3', '2'), ('2', '1', '3'),... Every posible combination\
print(list(permutations('123',2))) = \[('1', '2'), ('1', '3'), ('2', '1'), ('2', '3'), ('3', '1'), ('3', '2')] Every possible combination of length 2

**combinations**\
from itertools import **combinations** --> Generates all possible combinations without repeating characters (if "ab" existing, doesn't generate "ba")\
print(list(**combinations**('123',2))) --> \[('1', '2'), ('1', '3'), ('2', '3')]

**combinations_with_replacement**\
from itertools import **combinations_with_replacement** --> Generates all possible combinations from the char onwards(for example, the 3rd is mixed from the 3rd onwards but not with the 2nd o first)\
print(list(**combinations_with_replacement**('1133',2))) = \[('1', '1'), ('1', '1'), ('1', '3'), ('1', '3'), ('1', '1'), ('1', '3'), ('1', '3'), ('3', '3'), ('3', '3'), ('3', '3')]

# Decorators

Decorator that size the time that a function needs to be executed (from [here](https://towardsdatascience.com/decorating-functions-in-python-619cbbe82c74)):

```python
from functools import wraps
import time
def timeme(func):
  @wraps(func)
  def wrapper(*args, **kwargs):
    print("Let's call our decorated function")
    start = time.time()
    result = func(*args, **kwargs)
    print('Execution time: {} seconds'.format(time.time() - start))
    return result
  return wrapper

@timeme
def decorated_func():
  print("Decorated func!")
```

If you run it, you will see something like the following:

```
Let's call our decorated function
Decorated func!
Execution time: 4.792213439941406e-05 seconds
```

# Argument packing & unpacking

## Packing
```python
def sample(*args):
    print("Packed arguments:", args)

sample(1, 2, 3, 4, "geeks for geeks") # Packed arguments: (1, 2, 3, 4, 'geeks for geeks')

def sample(**kwargs):
    print("Packed keyword arguments:", kwargs)

sample(name="Anaya", age=25, country="India") # Packed keyword arguments: {'name': 'Anaya', 'age': 25, 'country': 'India'}
```

## Unpacking
```python
def addition(a, b, c):
    return a + b + c

num = (1, 5, 10)  
result = addition(*num) 
print("Sum:", result) # Sum: 16

def info(name, age, country):
    print(f"Name: {name}, Age: {age}, Country: {country}")

data = {"name": "geeks for geeks", "age": 30, "country": "India"}
info(**data) # Name: geeks for geeks, Age: 30, Country: India
```