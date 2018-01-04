# Chapter 16 - Coroutines

## Summary

## Important Quotes
> "When you start thinking of yield primarily in terms of control flow, you have the mindset to understand coroutines." - pg.463

> The advantage of using a coroutine is that total and count can be simple local variables: no instance attributes or closures are needed to keep the context between calls. - pg.469

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
