---
title: "[OS] Process, Thread"
date: 2022-07-05
categories: [Study, CS]
tags: [Process, Thread, Context-Switching]
layout: post
toc: true
---

Front Team과 협업을 하면서 가장 번거로웠던 작업은 OpenAPI를 만드는 작업이었다. 그러던 와중 API 개발만 해도 OpenAPI를 자동으로 만들어 준다는 **FastAPI라는 Python 프레임 워크**를 알게되었고, 현재 다니고 있는 회사의 백엔드 프레임 워크로 사용중이다.  

부끄러운 이야기이지만(..😰) 개발을 하면서 소스 코드 수준에서의 리팩토링이나 알고리즘 개선은 많이 다루었지만, **<u>FastAPI 프레임 워크의 특징 및 장점</u>**들에 대해서는 잘 알지 못하고 사용하였다.  

`잘 알지 못하고 배포한` 백엔드 서버는 결국 <u>Production 환경 배포 전 부하 테스트</u>에서 어마어마한 에러를 토해냈다. 개발자로서 `잘 알지 못하고` 사용했다는 것이 굉장히 부끄러웠고, 스스로에게 큰 실망을 했다. 그래서 <u>다시는 이러한 실수를 다시 반복하지 않기 위해</u>, 앞으로는 `다` 는 아니더라도 `잘` 알고 쓰고자 한다.

평소 CS 지식이 많이 부족했지만, 실질적으로 와닿는 부분이 없어 지나치곤 했다. 하지만, 많은 에러 경험 덕분(?)에 점점 더 근본적인 문제에 대해 고민하게 되었고, 현재 이 포스팅까지 쓰게 되었다. 이번 포스팅을 시작으로 CS 지식을 많이 습득하고자 한다. **첫 번째 주제는  `Process`와 `Thread` 로 선정했다.**

<br>

### 목차
1. [Process](#1-process)
2. [Thread](#2-thread)
3. [Context switching](#3-context-switching)


<br>

## 1. Process
---
프로세스를 설명하기 전, `프로그램`이란 무엇인지 간략하게 짚고 넘어가고자 한다.

> `프로그램` 이란?  

- 운영체제에 의해서 실행될 수 있는 실행 파일을 뜻한다.
- 즉, 코드가 구현되어 있는 파일을 프로그램으로 볼 수 있다
  - e.g. python file, python.exe, uvicorn 등  
  

> `프로세스` 란?  

- 운영체제 위에서 CPU, Memory를 사용하며 프로그램을 실행시키는 주체
- 각각의 프로세스는 `독립된 메모리 공간`을 할당 받는다. 따라서, 프로세스 하나가 죽더라도 다른 프로세스에는 영향이 가지 않는다.
- 여러 개의 프로세스가 돌아 갈 때는 `시분할`로 돌아가며, 프로세스 간 `Context switching` 은 느리다.
  > **프로세스**는 **스레드**와 달리 공유하는 영역이 없어서 캐시 데이터를 다 버리고 다시 캐시를 만드는 과정이 발생하기 때문에 `Context switching`이 스레드에 비해 느리다.  

<br>

![Process](/assets/img/study/cs/process.png)
*출처: https://gmlwjd9405.github.io*
- 프로세스는 각각 독립된 메모리 영역(Code, Data, Stack, Heap의 구조)을 할당받는다.
- 한 프로세스가 다른 프로세스 자원에 접근하려면 프로세스 간의 통신(IPC, inter-process communication)을 사용해야 한다.


<br>

## 2. Thread
---
> `스레드` 란?  

- `프로세스` 내에서 실제로 작업을 수행하는 주체를 의미함
- **모든 프로세스에는 한 개 이상의 `스레드`가 존재**하여 작업을 수행
- 두 개 이상의 스레드를 가지는 프로세스를 **멀티 스레드 프로세스(multi-threaded process)**라고 하며, 프로세스 내에서 동시에 여러 작업을 실행하는데 목적을 두고 있다.
- 스레드는 ThreadStack 영역을 제외한 나머지 영역(Code, Data, Heap)은 공유하고 있다. 즉, 스레드는 공유하는 영역이 많기 때문에 프로세스에 비하여 `Context switching`이 빠르다. 
- 그러나, <u>공유하는 전역 변수를 여러 스레드가 함께 사용하게되면 충돌이 발생</u>할 수 있기 때문에 스레드 간 통신에는 충돌 문제가 발생하지 않도록 동기화 문제를 잘 해결해야 한다.  

<br>

![Thread](/assets/img/study/cs/thread.png)
*출처 : https://gmlwjd9405.github.io*
- 각각의 스레드는 별도의 레지스터와 스택을 갖고 있지만, Heap 메모리는 서로 읽고 쓸 수 있다.
- 한 스레드가 프로세스 자원을 변경하면, 다른 이웃 스레드(sibling thread)도 그 변경 결과를 즉시 볼 수 있다.


## 3. Context switching
---
> `Context switching` 이란?  

- 문맥교환이라고도 하며, CPU가 실행하는 작업을 변경하는 것을 말한다.
- 즉, **동작 중인 프로세스가 대기**를 하면서 해당 <u>프로세스의 상태(Context)를 보관</u>하고, **대기하고 있던 다음 순서의 프로세스가 동작**하면서 이전에 보관했던 프로세스의 상태를 복구하는 작업을 말한다.
- 그렇다면, `Context switching`은 언제 발생할까?
  > 인터럽트(Interrupt)
    - CPU가 프로그램을 실행하고 있을 때, **실행 중인 프로그램 밖**에서 <u>예외 상황이 발생</u>하여 처리가 필요한 경우 CPU에게 알려 <u>작동이 중단되지 않고 예외 상황을 처리</u>할 수 있도록 하는 기능
  - `Context switching` 은 아래와 같은 `인터럽트(Interupt)` 요청이 왔을 때 발생한다.
    1. 입/출력 요청
    2. CPU 사용시간 만료
    3. 자식 프로세스 Fork
- `스케줄러`는 `Context switching`을 하는 주체로써 `Context switching`이 발생했을 때, 다음번 프로세스는 `스케줄러`가 결정한다.

- `Context switching` 은 인터럽트 상황에 대하여 프로세스가 유휴 상태로 대기하고 있는 것을 방지해 프로세스의 응답 시간을 단축할 수 있다.
- 하지만, 잦은 `Context swtiching` 발생은 오히려 오버헤드(Overhead) 비용을 발생시켜 성능을 떨어뜨린다.

![Context Switching](/assets/img/study/cs/context-switching.jpeg) 
*출처: https://www.crocus.co.kr/1364*
- 위 그림에서 프로세스 `P0`가 <u>실행 중인 상태(executing)에서 유휴 상태(idle)</u>가 될 때, 프로세스 `P1`이 곧바로 실행 상태로 되지 않고, <u>유휴 상태(idle)가 유지 되다가 변경</u>된다.
- 실행 상태로 즉각 변경되지 않는 이유는 프로세스 `P0`의 상태를 `PCB`에 저장하고 프로세스 `P1` 상태를 `PCB`에서 가져와야 하기 때문이다.
- 이 과정에서 CPU는 아무런 동작을 하지 못하게 되고, 이는 곧 성능 저하로 이어진다.


<br>

다음 포스팅에서는 멀티 프로그래밍, 멀티 태스킹, 멀티 스레딩, 멀티 프로세싱에 대해서 다루어 보도록 하겠다. 추가로, 현재 내가 사용중인 `Python` 의 `Process` , `Thread` 는 어떻게 구성되어 있는지도 알아볼 예정이다.


## Reference
- [멀티 프로세스와 멀티 스레드](https://frozenpond.tistory.com/124)  
- [Context switching](https://beststar-1.tistory.com/26#'%EC%9A%B4%EC%98%81%EC%B2%B4%EC%A0%9C,_%EC%BB%B4%ED%93%A8%ED%84%B0%EA%B5%AC%EC%A1%B0'_%EC%B9%B4%ED%85%8C%EA%B3%A0%EB%A6%AC%EC%9D%98_%EB%8B%A4%EB%A5%B8_%EA%B8%80)  
- [프로세스와 스레드 차이](https://gmlwjd9405.github.io/2018/09/14/process-vs-thread.html)