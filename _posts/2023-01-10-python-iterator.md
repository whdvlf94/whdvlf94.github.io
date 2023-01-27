---
title: "[Python] Iterator and Generator"
date: 2023-01-10
categories: [Programming, Python]
tags: [Iterator, Generator, Python]
layout: post
toc: true
---

<br>
이 번 포스팅에서는 **iterator**와 **generator**에 대해서 알아보도록 하겠다.

## 목차
1. [Iterator?](#1-iterator)
   1. [Iterator 만들기](#11-iterator-만들기)
   2. [iter](#12-iter)
   3. [next](#13-next)
2. [Generator?](#2-generator)
   1. [Generator 만들기](#21-generator-만들기)
   2. [yield from](#22-yield-from)


## 1. Iterator?
---
<br>
**<u>이터레이터(iterator)는 값을 차례대로 꺼낼 수 있는 객체(object) 또는 반복자</u>**를 말한다. for 반복문을 사용할 때 흔히 `range()` 를 활용해 연속된 숫자를 만들어 낸다고 생각하지만, 사실 숫자를 차례대로 꺼낼 수 있는 이터레이터가 내부에서 동작하고 있는 것이다.

이처럼 Python에서는 이터레이터만 생성하고 값이 필요한 시점이 되었을 때 값을 만드는 방식을 사용하고, 이를 **지연 평가(lazy evaluation)**라고 한다.

*문자열, 리스트, 딕셔너리, 세트 등 반복이 가능한 객체*는 내부 magic method에 `__iter__`  메서드가 포함되어 있다. 이 때, 반복이 가능한 객체에서 `__iter__` 를 호출하면 이터레이터 객체를 호출할 수 있다. 또한, `__next__`  메서드를 사용하면 반복 가능한 객체 내의 요소를 차례대로 꺼낼 수 있으며, 이터레이터 내 모든 요소가 다 호출되면 `StopIteration`  예외를 발생시킨다.

```python
it = [1,2,3].__iter__()

it.__next__()
1

it.__next__()
2
```
<br>

**딕셔너리와 세트는 반복 가능한 객체**이지만 <u>순서가 정해져 있지 않기 때문에</u>, 시퀀스 객체에 포함되지 않는다.

![반복 가능한 객체](/assets/img/programming/python/iterator.png){: width="500"}

### 1.1 Iterator 만들기
```python
class Counter:
    def __init__(self, stop):
        self.current = 0
        self.stop = stop

    def __iter__(self): #현재 인스턴스 반환
        return self

    def __next__(self):
        if self.current < self.stop:
            r = self.current
            self.current += 1
            return r
        else:
            raise StopIteration
```
위 객체는 리스트, 문자열, 딕셔너리, 세트, range 처럼 `__iter__` 를 호출해줄 반복 가능한 객체(iterable)가 없으므로 `__iter__` 메세드에서 현재 인스턴스를 반환 해준다.


```python
class Counter:
    def __init__(self, stop):
        self.current = 0
        self.stop = stop

    def __getitem__(self, index):
        if index < self.stop:
            return index
        else:
            raise IndexError
```
위 과정을 조금 더 간단하게 나타낼 수 있는 방법으로는 `__getitem__` 메서드를 사용하는 방법이 있고, 이를 구현하면 인덱스로 접근할 수 있는 이터레이터가 생성된다. `__getitem__` 메서드를 사용하는 경우 `__iter__`, `__next__` 를 생략할 수 있다.

python 내장 함수에서는 `iter()`, `next()`를 각각 지원한다. 

### 1.2 iter
iter는 반복을 끝낼 값을 지정하면 특정 값이 나올 때 반복을 끝낸다.
> iter(호출가능한 객체, 반복을 끝낼 값)

이 때, **기존과 다른 점으로는 반복 가능한 객체가 아닌 호출가능한 객체를 입력받다는 것**이 있다
```python
import random
it = iter(lambda : random.randint(0,5), 2)
next(it)
3
next(it)
1
next(it)
Traceback (most recent call last):
  File "<pyshell#37>", line 1, in <module>
    next(it)
StopIteration
```

### 1.3 next
next는 기본값을 지정할 수 있다. **기본값을 지정하면 반복이 끝나더라도 `StopIteration`이 발생하지 않고 기본값을 출력**한다.
> next(반복 가능한 객체, 기본값)

```python
it = iter(range(2))
next(it, 10)
0
next(it, 10)
2
next(it, 10)
10
next(it, 10)
10
```

<br>
## 2. Generator?
---

**제너레이터(generator)는 이터레이터를 생성해주는 함수**이다. 이터레이터(iterator)는 클래스 내부에 `__iter__`, `__next__` 또는 `__getitem__`을 구현해야 하지만, **제너레이터는 함수 안에서 `yield`라는 키워드만 사용**하면 된다.

```python
def number_generator():
    yield 0
    yield 1
    yield 2

for i in number_generator():
    print(i)

0
1
2
```

먼저 제너레이터 객체를 만든 후, `next()`를 호출하면 제너레이터 안의 yield 0이 실행되어 숫자 0을 전달한 뒤 바깥의 코드가 실행되도록 양보한다. `yield`는 함수를 끝내지 않은 상태에서 값을 함수 바깥으로 전달할 수 있다.

![반복 가능한 객체](/assets/img/programming/python/generator.png){: width="500"}

### 2.1 Generator 만들기

```python
def number_generator(stop):
    n = 0
    while n < stop:
        yield n
        n += 1

def i in number_generator(3):
    print(i)
```

`yield`<u>에서는 변수 뿐만 아니라 함수도 호출할 수 있다.</u> 이 때, `yield`에서 함수(메서드)를 호출하면 해당 함수의 반환값을 바깥으로 전달한다.

```python
def upper_generator(x):
    for i in x:
        yield i.upper()    # 함수의 반환값을 바깥으로 전달
 
fruits = ['apple', 'pear', 'grape', 'pineapple', 'orange']
for i in upper_generator(fruits):
    print(i) #모두 대문자로 출력
```

### 2.2 yield from
`yield from`을 사용하면 반복 가능한 객체를 대상으로 요소를 한 개씩 바깥으로 전달 할 수 있다.

```python
def number_generator():
    x = [1, 2, 3]
    yield from x    # 리스트에 들어있는 요소를 한 개씩 바깥으로 전달
 
for i in number_generator():
    print(i)
```

또한, **yield from에는 제너레이터 객체를 지정하는 것도 가능**하다.

number_generator(3)은 숫자를 세개 만들어 내므로 `yield from` 역시 숫자를 세 번 바깥으로 전달한다.
```python
def number_generator(stop):
    n = 0
    while n < stop:
        yield n
        n += 1
 
def three_generator():
    yield from number_generator(3)    # 숫자를 세 번 바깥으로 전달
 
for i in three_generator():
    print(i)
```

> 리스트 표현식을 `[]`가 아닌 `()`묶으면 제너레이터 표현식이 된다. 이는 필요할 때 마다 요소를 만들어내므로 메모리를 절약할 수 있다.
{: .prompt-info} 
