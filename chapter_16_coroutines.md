# Chapter 16 - Coroutines

## Summary

## Important Quotes
> "When you start thinking of yield primarily in terms of control flow, you have the mindset to understand coroutines." - pg.463

> The advantage of using a coroutine is that total and count can be simple local variables: no instance attributes or closures are needed to keep the context between calls. - pg.469

> The main feature of yield from is to open a bidirectional channel from the outermost caller to the innermost subgenerator, so that values can be sent and yielded back and forth directly from them, and exceptions can be thrown all the way in without adding a lot of exception handling boilerplate code in the intermediate coroutines. This is what enables coroutine delegation in a way that was not possible before. - pg.478

> It’s arguably more natural to implement a continuous simulation using threads to account for actions happening in parallel in real time. On the other hand, coroutines offer exactly the right abstraction for writing a DES. - pg.490

## Important functions related to coroutine

1. `<coroutine>.send(value)`
2. `next(<coroutine>)`
3. `<coroutine>.throw(Exception)`
4. `<coroutine>.close()`

## Four states of Coroutines

**1. 'GEN_CREATED':**
Waiting to start execution.
**2. 'GEN_RUNNING':**
Currently being executed by the interpreter.
**3. 'GEN_SUSPENDED':**
Currently suspended at a yield expression.
**4. 'GEN_CLOSED':**
Execution has completed.

## Basic behavior of Generator used as a Coroutine

```python3
def simple_coroutine():
  print('-> coroutine started')
  x = yield
  print('-> coroutine received:', x)
  y = x + 10
  print('y:', y)
  z = yield
  print('-> coroutine received:', z)
```
```
>>>coroutine = simple_coroutine()
>>>next(coroutine)
-> coroutine started
>>>coroutine.send(10)
-> coroutine received:10
y: 20
>>>coroutine.send(20)
-> coroutine received:20
---------------------------------------------------------------------------
StopIteration                             Traceback (most recent call last)
<ipython-input-7-072e57f4c8a2> in <module>()
----> 1 coroutine.send(20)

StopIteration:
```

Basically, the function executes to where yield is, then through send, it can receive new inputs. It's like creating multiple stages inside a function - control flow.

## Example: Coroutine to calculate average / Priming Coroutines
```python3
def averager():
    total = 0.0
    count = 0
    average = None
    while True:
        term = yield average
        total += term
        count += 1
        average = total/count
```

When you try to use this function, you will realize that you cannot send a value immediately. You need to advance the coroutine to `yield average` initially, then repeat send.

To do this automatically, we can use decorators.

```python3
def coroutine(func):
"""Decorator: primes `func` by advancing to first `yield`"""
  @wraps(func)
  def primer(*args,**kwargs):
    gen = func(*args,**kwargs)
    next(gen)
    return gen
  return primer
```

Frameworks usually come with these types of decorators. For example, **tornado.gen decorator.**.
`yield from` syntax automatically primes the coroutine calling it.

## Coroutine Termination and Exception Handling
On the `averager()` function above, if we send a string value, `TypeError` will happen because
`total += "somestring"` is not possible. In this case, the coroutine will **terminate.**

However, we can use the regular `except` to keep it going.

```python3
def averager():
    total = 0.0
    count = 0
    average = None
    while True:
        term = yield average
        try:
            total += term
        except TypeError:
            print("TypeError. Continuing...")
            continue
        count += 1
        average = total/count
```


## Returning a value
#### Before
```python3
def averager():
total = 0.0
count = 0
average = None
while True:
  term = yield
  if term is None:
    break
  total += term
  count += 1
  average = total/count
return Result(count, average)
```
```
>>> coro_avg = averager()
   >>> next(coro_avg)
   >>> coro_avg.send(10)
   >>> coro_avg.send(30)
   >>> coro_avg.send(6.5)
   >>> coro_avg.send(None)
   Traceback (most recent call last):
...
StopIteration: Result(count=3, average=15.5)
```
- `if term is None` is key - you have to terminate normally somehow, and here, it is done with if a blank send is given.
- But, value is returned with StopIteration Exception.

To solve this problem, it was required to use `try...except` to catch the result on the exception. Then, *yield from* comes to the rescue.

## Using yield from
**!Warning: this is where things get real**
- Similar to `await` in other languages(asyncio uses `await` too! we'll find out later.)

#### Brief overview
The two following functions do the same thing:
```
def gen():
  for c in 'AB':
    yield c
  for i in range(1, 3):
    yield i
>>> list(gen())
['A', 'B', 1, 2]
```
```
def gen():
  yield from 'AB'
  yield from range(1, 3)
>>> list(gen())
['A', 'B', 1, 2]
```
If yield from was just this, it would be very boring.
Well what is 'just this'?

`yield from x` calls `iter(x)` to obtain an iterable from the iterator `x`. `x` then, can be a generator function.

#### Delegating generator & subgenerator & caller
```
from collections import namedtuple
Result = namedtuple('Result', 'count average')

# the subgenerator
def averager():
  total = 0.0
  count = 0
  average = None
  while True:
    term = yield
    if term is None:
      break
    total += term
    count += 1
    average = total/count
return Result(count, average)

# the delegating generator
def grouper(results, key):
  while True:
    results[key] = yield from averager()

# the client code, a.k.a. the caller
def main(data):
  results = {}
  for key, values in data.items():
    group = grouper(results, key)
    next(group)
    for value in values:
      group.send(value)
      group.send(None) # important!

  # print(results)  # uncomment to debug
  report(results)
```

## Coroutine Use Case : Discrete Event Simulation
Discrete Event Simulation: Type of simulation where a system is modeled as a sequence of events. <br>
Think of turn based games like chess. Each "play" is a single move of a chess piece disregarding the time it took to make that play. Or perhaps think of taxis, where each state is determined by the passenger in the taxi disregarding the duration of their stay.

The following example utilizes SimPy. It is not intended to be an introduction to the library as much as it is to develop an intuition of programming concurrent actions with coroutines.
