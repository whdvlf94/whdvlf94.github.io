---
title: "[Python] __hash__ 및 __eq__를 사용한 Python 해싱 및 동등성 이해"
date: 2023-01-26
categories: [Programming, Python]
tags: [Python, built-in]
layout: post
toc: true
---


## 목차
1. [__eq__ 연산자](#1-__eq__-연산자와-is)
2. [__hash__ 연산자](#2-__hash__-연산자)
3. [Reference](#reference)


## 1. `__eq__` 연산자와 `is`
---

python에서 `__eq__` 연산자는 `==` 연산자를 오버로딩(Overloading)하는 방법이다.
```python
class SomeClass:
    ...
    def __eq__(self, other): #매개변수 재정의에 따른 오버로딩
        # return True if this object
        # is equal to other and False
        # otherwise.
    ...
```

만약, 사용자 지정 클래스에서 `__eq__` 메서드를 구현하지 않는다면 해당 클래스에서 생성된 객체는 `is` 연산자와 `==` 연산자가 동일하게 동작한다. **즉, 객체가 동일한지 확인만 하고 클래스의 속성은 신경쓰지 않는다.**

```python
class MyClass:
    def __init__(self, a, b):
        self.a = a
        self.b = b

a = MyClass(5,10)
b = MyClass(5,10)

assert a != b #True
assert a is not b #True


# __eq__ 메서드 구현
class MyCustomClass:
    def __init__(self, a, b):
        self.a = a
        self.b = b

    def __eq__(self, other):
        return self.a == other.a and self.b == other.b


a = MyCustomClass(5,10)
b = MyCustomClass(5,10)

assert a == b #True
assert a is not b #True
```

## 2. `__hash__` 연산자
---
Python의 `hash()` 함수는 `built-in` 함수이며 개체에 해시 값이 있는 경우 해당 개체의 해시 값을 반환한다. **이 때, 해시 값은 딕셔너리(`dict`)를 보면서 빠르게 키를 비교하는 데 사용되는 정수이다.**

Python의 딕셔너리(dictionary)는 `hash map` 이며, set 또는 frozenset도 이 `hash map`을 사용하여 구현된다. Python에서 `__hash__` 함수는 `set`과 `dict`에서 Key를 해싱하기 위한 해시함수로 사용되고 있으며, `__eq__`와 마찬가지로 클래스에 추가할 수 있는 메서드이다.
```python
class Example:
    def __init__(self, a):
        self.a = a

    def __hash__(self):
        return self.a


ex = Example(3)
hash(ex) #3
```

**`hash()` 함수의 속성**
- `hash()`를 사용하여 **해시된 개체는 되돌릴 수 없다(단방향성)**.
- `hash()`는 **불변 객체에 대해서만 해시 값을 반환**하므로 가변/불변 객체를 확인하는 지표로 사용할 수 있다.
  - 가변(mutable) 객체: list, set, dict
  - 불변(immutable) 객체: int, float, bool, tuple, string, unicode
- **두 객체가 같다면(`__eq__`가 True 라면) 동일한 `__hash__` 값을 가져야 한다**(<u>그러나 반대의 경우는 반드시 참은 아니다</u>)
- `__hash__`를 정의하지 않고 `__eq__`만 정의한다면, 해당 개체는 **해시화 할 수 없다.**
- `__eq__` 메서드가 가변/변경 가능한 속성을 사용하는 경우 **`__hash__` 메서드를 정의해서는 안된다.**

<br>

아래와 같은 상수 값을 가지는 해시는 **해시 테이블에 단일 버킷만 존재하여 충돌을 야기**한다. 즉, **일정한 해시를 사용하면 단일 버킷의 항목이 반복되고 일치 항목이 발견될 때 까지 속성 값을 비교**해야 하기 때문에 <u>해시 테이블의 데이터 삽입 시간 복잡도는</u> `O(n^2)`<u>이 된다.</u>
```python
class Example:
    def __init__(self, a):
        self.a = a
        
    def __eq__(self, other):
        return self.a == other.a
    
    def __hash__(self):
        return 1

a = Example(1)
b = Example(2)

obj = { a: 10, b: 20 }
# It works:
print(obj[Example(1)])
# => 10
print(obj[Example(2)])
# => 20
```

위 상황을 해시 테이블로 도식화 해보면 아래와 같이 나타날 것이다. 즉, **상수 값을 가지고 있는 해시 테이블의 경우 인덱스 값이 동일하기 때문에 `__eq__` 메서드를 활용하여 속성 값이 일치하는지 모두 확인**해야 한다.

| hash value | Actual value |
|------------|--------------|
|      1     |    [a, b]    |
|      2     |     NULL     |
|      3     |     NULL     |

<br>
<br>
`hash()` 함수에 대해 간략하게 작성하려고 했는데, 막상 자료를 찾다보니 자료구조에 대한 부족함을 많이 느꼈다. 따라서, 다음 포스팅에서는 이번 글을 쓸 때에 잘 이해가 되지 않았던 `해시 테이블`에 대해서 작성하도록 하겠다 !
<br>

## Reference
---
[Understanding Hashing and Equality in Python with __hash__ and __eq__](https://levelup.gitconnected.com/understanding-hashing-and-equality-in-python-with-hash-and-eq-12f6da79e8ad)<br>
[python __hash__](https://docs.python.org/3.9/reference/datamodel.html#object.__hash__)<br>
[hash 테이블 동작 원리](https://stackoverflow.com/a/32280685)