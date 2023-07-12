---
title: "[Python] Python 비동기 asyncio 사용하기"
date: 2023-05-05
categories: [Programming, Python]
tags: [Python, asyncio, GIL]
layout: post
toc: true
---

## 목차
1. [Python WAS](#1-python-was)
   1. [CGI, FastCGI](#cgi-fastcgi)
   2. [WSGI](#wsgi)
   3. [ASGI](#asgi)
2. [Python 비동기란?](#2-python-비동기란)
   1. [코루틴](#21-코루틴)
   2. [네이티브 코루틴](#22-네이티브-코루틴)
3. [GIL 이란?](#3-gil-이란)
4. [왜 asyncio를 사용해야 하는가?](#4-왜-asyncio를-사용해야-하는가)
5. [asyncio 사용하기](#5-asyncio-사용하기)
6. [async/await](#6-asyncawait)
   1. [asyncio.create_task()](#61-asynciocreat_task)
   2. [asyncio.gather()](#62-asynciogather)
   3. [loop.run_inexecutor()](#63-looprun_in_executor)
7. [Reference](#reference)

<br>

# 1. Python WAS?
---

Python에서는 Tomcat과 같은 WAS가 별도로 존재하지 않는다. 그렇다면 어떤 방법으로 웹서버와 Python 애플리케이션을 연결할 수 있을까?

<br>

### CGI, FastCGI
> Common Gateway Interface

외부 애플리케이션과 웹 서버(Nginx, Apache 등)를 연결해주는 표준화된 프로토콜. CGI는 클라이언트의 요청이 발생할 때 마다 프로세스를 추가로 생성하고 삭제하기 때문에 오버헤드와 성능 저하의 원인이 되었다.

![CGI](/assets/img/programming/asyncio/cgi.png){: width="600"}

따라서, 이를 개선하고자 FastCGI가 등장하였다. FastCGI는 몇 번의 요청이 들어와서 하나의 프로세스만을 가지고 처리한다. 즉, 메모리에 단 하나의 프로그램만을 적재하여 재활용하기 때문에 CGI에 비하여 오버헤드가 월등하게 감소한다.

<br>

**Java의 Tomcat 또한 FastCGI Web Server + FastCGI 방식을 채택하고 있다**

![FastCGI](/assets/img/programming/asyncio/fastcgi.png)

하지만, Python에서는 이러한 WAS별도로 존재하지 않는다. 따라서, Python 만의 별도 게이트웨이 인터페이스를 만들었는데, 그게 바로 `WSGI`와 `ASGI` 다.


### WSGI
> Web Server Gateway Interface

WSGI는 Python 애플리케이션, 웹 서버가 통신하기 위한 인터페이스로써 CGI 패턴을 모태로 하여 만들어졌다. WSGI는 모든 요청을 한 프로세스에서 받으며,  각 요청을 콜백(callback)을 받아 처리하게 된다. **즉, WSGI는 웹 서버와 애플리케이션 사이에서 인터페이스(미들웨어) 역할을 한다.**

대표적인 **WSGI Middleware로는 gunicorn, uWSGI**가 있으며, **WSGI Application으로는 Flask, django**가 있다.

![WSGI](/assets/img/programming/asyncio/wsgi.png)

그러나, WSGI는 **Synchronous**하게 작동하기 때문에 동시에 많은 Request를 처리하는데 한계가 존재함

> `Celery`, `Queue`를 이용하여 비동기적 요청에 대한 성능 향상을 할 수 있다.
{: .prompt-warning} 

### ASGI
> Asynchronous Server Gateway Interface

현대 웹 서비스는 점점 더 많은 양의 트래픽 처리를 요구하고 있기에, Synchronous하게 동작하는 방식으로 처리하는 데에는 한계가 있다. 이는 ASGI가 등장하게 된 배경이 되었다.

ASGI는 WSGI와 비슷한 구조를 가지나 기본적으로 모든 요청을 **Asynchronous**로 처리하며, WSGI에서 지원하지 않는 Websocket, HTTP 2.0을 지원한다. 또한, ASGI는 WSGI와 호환된다(<u>ASGI는 WSGI의 상위 버전</u>)

대표적인 **ASGI Server로는 uvicorn**가 있으며, uvicorn은 내장 모듈로 uvloop을 사용한다. uvloop는 Javascript V8에서 사용되는 비동기 모듈을 사용하고 있으며, Cython 기반으로 C++언어로 작성되어 Go, Node JS 에 준하는 성능을 제공한다고 한다. **ASGI Middleware는 WSGI와 동일한 gunicorn을 사용**하는 것을 볼 수 있는데, ASGI에서 gunicorn은 프로세스 매니저로서 동작하게 된다(NodeJs에서의 pm2의 역할)

> Django 3.0, Falcorn 3.0 부터 ASGI를 지원한다.
{: .prompt-info}

![ASGI](/assets/img/programming/asyncio/asgi.png)

<br>

# 2. Python 비동기란?
---

Python2와 Python3를 비교했을 때,  Python3에서 가장 돋보이는 특징은 비동기 프로그래밍 지원이라고 할 수 있다. Python 3.4 버전부터는 asyncio 패키지가 추가되었고, Python 3.5 버전 부터는 네이티브 코루틴(native coroutine) 지원을 위한 async/await 키워드가 추가되었다.

> Python에서는 제너레이터 기반의 코루틴과 구분하기 위해 `async def`로 만든 코루틴은 `네이티브 코루틴(native coroutine)` 이라고 한다.
{: .prompt-info}

## 2.1 코루틴
```python
def add(a, b):
    c = a + b    # add 함수가 끝나면 변수와 계산식은 사라짐
    print(c)
    print('add 함수')
 
def calc():
    add(1, 2)    # add 함수가 끝나면 다시 calc 함수로 돌아옴
    print('calc 함수')
 
calc()
```
위 소스 코드에서 `calc` 함수와 `add` 함수는 `메인 루틴(main routine)` 과 `서브 루틴(sub routine)` 관계를 가지고 있으며, 도식화 하면 아래 그림과 같이 나타낼 수 있다.

![subroutine](/assets/img/programming/asyncio/subroutine.png){: width="500"}

Python `코루틴(coroutine)`은 `메인 루틴`과 `서브 루틴`처럼 종속된 관계가 아닌 서로 대등한 관계이며, 특정 시점에 상대방의 코드를 실행한다.

**코루틴은 일반 함수와 달리 종료되지 않은 상태에서 메인 루틴의 코드를 실행한 뒤 다시 돌아와서 코루틴의 코드를 실행할 수 있으며, 종료되지 않은 상태로 대기 하기 때문에 코루틴의 내용도 계속 유지된다.**

```python
def sum_coroutine():
    total = 0
    while True:
        x = (yield total)    # 코루틴 바깥에서 값을 받아오면서 바깥으로 값을 전달
        total += x
 
co = sum_coroutine()
print(next(co))      # 0: 코루틴 안의 yield까지 코드를 실행하고 코루틴에서 나온 값 출력
 
print(co.send(1))    # 1: 코루틴에 숫자 1을 보내고 코루틴에서 나온 값 출력
print(co.send(2))    # 3: 코루틴에 숫자 2를 보내고 코루틴에서 나온 값 출력
print(co.send(3))    # 6: 코루틴에 숫자 3을 보내고 코루틴에서 나온 값 출력
```

![coroutine](/assets/img/programming/asyncio/coroutine.png){: width="500"}

## 2.2 네이티브 코루틴
비동기 코루틴은 기본적으로 `def` 앞에 `async`를 붙여서 사용한다. 그리고 내부에서 다른 비동기 작업을 호출하게 되면 `await`를 붙여야 한다. 또한 `await`는 기다리는 동안 스케줄링 된 다른 작업으로 전환이 가능해야 하므로 `async def` <u>로 정의된 블럭 내에서만 사용할 수 있다.</u>

`async def` 를 써서 정의하는 함수를 **코루틴 함수**라 하며 코루틴 함수를 호출하면 (비동기) 코루틴 객체를 얻을 수 있다. `await` 표현식은 `coroutine`의 실행을 일시 중지하며, 해당 작업이 완료될 때 까지 기다린다.

```python
import asyncio

async def main():
  print("hello")
  await asyncio.sleep(1)
  print("world")
  
asyncio.run(main())
```

# 3. GIL 이란?
---

Python은 GIL(Global Interpreter lock) 이라는 규칙이 존재한다. GIL을 이해하려면 먼저 Python 인터프리터가 무엇인지 알아야 한다. Python 인터프리터란, Python으로 작성된 코드를 한 줄씩 읽으면서 실행하는 프로그램을 뜻한다. 그러면 본격적으로 Python GIL 개념에 대해서 알아보도록 하겠다. [Python 위키](https://wiki.python.org/moin/GlobalInterpreterLock)에 따르면 GIL 정의는 다음과 같다.
> In CPython, the global interpreter lock, or GIL, is **a mutex that protects access to Python objects, preventing multiple threads from executing Python bytecodes at once.** This lock is necessary mainly because CPython's memory management is not thread-safe.

<br>

**해석하자면, 한 프로세스 내에서 Python 인터프리터는 한 시점에 하나의 쓰레드에 의해서만 실행될 수 있다. 즉, Python에서는 멀티 쓰레드를 사용하더라도 병렬처리가 불가능하다.**

> `multiprocessing` 패키지를 사용하면 병렬 처리가 가능하다
{: .prompt-warning}

![GIL](/assets/img/programming/asyncio/gil.png){: width="500"}


# 4. 왜 asyncio를 사용해야 하는가?
---

Python 에서는 GIL으로 인해 연산(CPU)의 경우 일반적으로 멀티 스레드가 싱글 스레드 보다 느리다. 하지만, 연산 작업이 아닌 **파일 읽고 쓰기, HTTP 통신 대기, DB Query**와 같은 **Blokcing I/O**의 경우에 `asyncio` **모듈을 사용하면 이벤트 루프(event loop)를 사용하여 비동기 작업을 처리하기 때문에 스레드를 사용하지 않는다.** 즉, 비동기 작업을 처리하는 동안 내부 이벤트 루프에서는 GIL에 의한 제약이 없기 때문에 대기 시간 동안 다른 작업을 처리할 수 있어 높은 처리량과 속도를 얻을 수 있다.

> `asyncio` 모듈은 기본적으로 단일 스레드에서 실행되며, 모든 작업이 이벤트 루프에서 처리된다. `asyncio`에서 사용되는 스레드는 스레드 풀에서 사용하는 스레드가 아닌, `asyncio` 내부적으로 관리되는 스레드를 사용한다.
{: .prompt-info}

# 5. asyncio 사용하기
---

`asyncio` (Asynchronous I/O)는 비동기 프로그래밍을 위한 모듈이며 **단일 스레드(Single Thread)를 사용**하여 CPU 작업과 I/O를 병렬로 처리 즉, **동시(concurrency)**에 실행할 수 있게 해준다. 이 때, **단일 스레드 동시성 모델(single-threaded concurrency model)**을 위해 `asyncio` 패키지는 **이벤트 루프(Event loop)**라는 구조를 사용하고 있다.
> 단일 스레드 동시성 모델에서는 항상 단일 스레드로 동작하며, I/O Bound 작업을 만나면 OS의 이벤트 알림 시스템에 전달하고 다른 작업(코드)를 실행한다. I/O Bound 바운드 작업을 추적하기 위해 asyncio 패키지는 이벤트 루프를 사용한다.

<br>

이벤트 루프의 동작 원리는 다음과 같다.
1. 먼저 메인 스레드(**Main Thread**)는 태스크(**Task**)를 태스크 큐(**Task Queue**)에 제출한다.
2. 이벤트 루프(**Event Loop**)는 태스크 큐를 지속적으로 모니터링하고 I/O 작업들이 나타날 때 까지 작업을 실행한다. I/O 작업이 발생하는 경우 이벤트 루프는 작업을 일시 중지하고 **OS**에 넘긴다.
3. 이벤트 루프는 완료된 I/O 작업을 확인한다. 작업이 완료되면 OS가 프로그램에 알려, 이벤트 루프가 중지되지 않은 작업을 실행한다.
4. 위 과정을 작업 대기열이 비워질 때까지 반복한다.

> Python 3.7 이후 asyncio 패키지는 이벤트 루프를 자동으로 관리할 수 있는 기능을 제공하므로 하위 수준 API를 처리할 필요가 없다.
{: .prompt-info}

![event-loop](/assets/img/programming/asyncio/eventloop.png){: width="500"}

# 6. async/await
---

Python에서 `coroutine`은 return에 도달하기 전에 실행을 일시 중지할 수 있는 함수이며, 일정 시간 동안 다른 coroutine에 간접적으로 제어를 전달할 수 있다.

Python에서 `coroutine`을 만들고 일시 중지하기 위해서는 `async`, `await` 키워드를 사용할 수 있다.

> `async`는 코루틴을 생성하고, `await`는 코루틴을 일시중지한다.
{: .prompt-info}

Python 3.7 이상 부터 `asyncio.run()` 함수를 통해 **이벤트 루프를 자동으로 생성**하고 **코루틴을 실행한 후 닫을 수 있다.** `asyncio.run()` 함수는 프로그램의 다른 코루틴 또는 함수를 호출할 수 있는 **하나의 코루틴만 실행**한다. 

`await` 키워드는 코루틴의 실행을 중단시킨다. 즉, `await` 키워드를 사용하면 해당 코루틴이 끝나고 결과 값을 리턴할 때 까지 대기하게 된다. **이 때, await 키워드는 반드시 코루틴 내부에서 사용해야 한다.**

```python
import asyncio


async def square(number: int) -> int:
    return number*number


async def main() -> None:
    x = await square(10)
    print(f'x={x}')

    y = await square(5)
    print(f'y={y}')

    print(f'total={x+y}')

asyncio.run(main())

============================
x=100
y=25
total=125
```

## 6.1 asyncio.creat_task()

```python
import asyncio
import time


async def call_api(message, result=1000, delay=3):
    print(message)
    await asyncio.sleep(delay)
    return result


async def main():
    start = time.perf_counter()

    price = await call_api('Get stock price of GOOG...', 300)
    print(price)

    price = await call_api('Get stock price of APPL...', 400)
    print(price)

    end = time.perf_counter()
    print(f'It took {round(end-start,0)} second(s) to complete.')

asyncio.run(main())

================================
Get stock price of GOOG...
300
Get stock price of APPL...
400
It took 6.0 second(s) to complete.
```

코드를 도식화 하면 아래와 같이 표현할 수 있다. 위 코드에서는 **코루틴을 이벤트 루프에 추가하지 않고, 직접 호출하는 방식**으로 작성되어 있다. 즉, 코루틴 객체를 얻어 await 키워드를 사용하는 방식으로 결과를 얻고 있다.

![coroutine-process](/assets/img/programming/asyncio/coroutine-process.png)

위 예제는  코루틴 객체를 얻어 `await` 키워드를 사용하여 실행한 결과이다(<u>이벤트 루프에 넣지 않았음</u>). 이는 `async` , `await` 키워드를 사용하여 비동기 방식으로 코드를 작성했지만, 동시에 실행시키지 않았다는 것을 의미한다. 

**여러 동작들을 동시에 실행시키기 위해서는 태스크를 활용해야 한다.** 태스크는 가능한 빨리 이벤트 루프에서 태스크가 실행되도록 코루틴을 예약하는 코루틴 wrapper이다.

스케줄링 및 실행은 non-blocking 방식이기 때문에 작업을 생성하고 작업이 실행되는 동안 다른 코드를 실행할 수 있다. **즉, 위 예제 방식과는 달리 여러 태스크를 생성하고 동시에 이벤트 루프에서 실행되도록 스케줄링 한다는 차이점이 있다.**

태스크를 생성하기 위해서 `asyncio.create_task()` 에 코루틴을 전달해야 하며, `create_task()` 함수는 태스크 객체를 반환한다. 

```python
import asyncio
import time


async def call_api(message, result=1000, delay=3):
    print(message)
    await asyncio.sleep(delay)
    return result


async def main():
    start = time.perf_counter()

    task_1 = asyncio.create_task(
        call_api('Get stock price of GOOG...', 300)
    )

    task_2 = asyncio.create_task(
        call_api('Get stock price of APPL...', 300)
    )

    price = await task_1
    print(price)

    price = await task_2
    print(price)

    end = time.perf_counter()
    print(f'It took {round(end-start,0)} second(s) to complete.')


asyncio.run(main())

================================
Get stock price of GOOG...
Get stock price of APPL...
300
300
It took 3.0 second(s) to complete.
```

![asyncio-task-process](/assets/img/programming/asyncio/asyncio-task-process.png)

> `asyncio.run()` 함수에 의해 이벤트 루프가 닫히기 전에 태스크가 완료될 수 있도록 `await` 키워드를 잘 사용해야 한다.
{: .prompt-info}

## 6.2 asyncio.gather()
`asyncio.gather()` 함수를 사용하면 여러 비동기 작업을 한 번에 실행하고 결과를 얻을 수 있다.

```python
gather(*aws, return_exceptions=False) -> Future[tuple[()]]
```

`asyncio.gather()` 함수는 두 개의 파라미터를 받고 있다. 
- `aws`: `aws` 는 awaitable 객체의 묶음이다. `aws`의 객체가 코루틴인 경우 `asyncio.gather()` 함수는 자동으로 태스크를 예약한다
- `return_exceptions`: `return_exceptions`는 `awaitable` 객체에서 예외가 발생하면 실행을 취소하지 않고, 그 다음 태스크를 실행하도록 하는 옵션이다. **해당 옵션 값을 True 로 바꾸면, awaitable 객체에서 예외가 발생할 경우 해당 객체만 예외처리 되며 나머지 작업은 정상적으로 실행된다.**
  
> `asyncio.gather()`는 awaitable 객체를 함수에 전달한 것과 동일한 순서의 결과를 반환한다.
{: .prompt-info}

```python
import asyncio


class APIError(Exception):
    def __init__(self, message):
        self._message = message

    def __str__(self):
        return self._message


async def call_api(message, result, delay=3):
    print(message)
    await asyncio.sleep(delay)
    return result


async def call_api_failed():
    await asyncio.sleep(1)
    raise APIError('API failed')


async def main():
    a, b, c = await asyncio.gather(
        call_api('Calling API 1 ...', 100, 1),
        call_api('Calling API 2 ...', 200, 2),
        call_api_failed(),
        return_exceptions=True
    )
    print(a, b, c)


asyncio.run(main())

==============================
Calling API 1 ...
Calling API 2 ...
100 200 API failed
```

## 6.3 loop.run_in_executor()

Python 3.7 이후로는 `asyncio`, `async/await`가 추가되었기 때문에 I/O Bound된 작업을 단일 스레드에서 비동기로 처리할 수 있는 방법이 생겼다. 하지만, 대부분의 Python 내장 라이브러리 함수들은 `coroutine` 이 아닌 일반 함수들이며, 이들은 모두 `blocking` 방식으로 동작한다는 문제가 있다.

따라서, asyncio 패키지는 동기 함수를 비동기로 동작할 수 있도록 이벤트 루프에서 `run_in_executor()` 함수를 제공한다([공식 Docs](https://docs.python.org/ko/3/library/asyncio-eventloop.html#asyncio.loop.run_in_executor)).

`run_in_executor()`함수는 `blocking`함수 콜 자체를 I/O Bound 작업으로 보고, 이를 기다리는 동안 일시 중지하는 비동기 코루틴으로 구현한 것이다.

```python
import time
import requests
import asyncio                       
import nest_asyncio
nest_asyncio.apply()

async def sync_time(i):
  loop = asyncio.get_event_loop()
  await loop.run_in_executor(None, time.sleep, i)

async def main() :
    await asyncio.gather(
        sync_time(1),
        sync_time(2),
        sync_time(3),
    )    

print(f"stated at {time.strftime('%X')}")
start_time = time.time()
asyncio.run(main())
finish_time = time.time()
print(f"finish at {time.strftime('%X')}, total:{finish_time-start_time} sec(s)")

========================================
stated at 01:36:03
finish at 01:36:06, total:3.0150146484375 sec(s)
```

# Reference
---
[Python WSGI, ASGI](https://blog.neonkid.xyz/249)

[Concurrency and async](https://fastapi.tiangolo.com/async/#asynchronous-code)

[Coroutine & Task](https://docs.python.org/ko/3/library/asyncio-task.html)

[Async I/O](https://realpython.com/async-io-python/#async-io-design-patterns)

[Python Event Loop](https://www.pythontutorial.net/python-concurrency/python-event-loop/)

[Nonblocking asynchrous coroutine](https://soooprmx.com/python-asycnio-%ec%97%90%ec%84%9c-%eb%9f%b0%eb%a3%a8%ed%94%84%eb%a5%bc-%ea%b8%b0%eb%b0%98%ec%9c%bc%eb%a1%9c-non-blocking-%ec%bd%94%eb%93%9c-%ec%9e%91%ec%84%b1%ed%95%98%ea%b8%b0/)