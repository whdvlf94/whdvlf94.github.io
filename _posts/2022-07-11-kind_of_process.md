---
title: "[OS] 멀티 프로그래밍, 멀티 태스킹, 멀티 프로세싱, 멀티 스레딩"
date: 2022-07-11
categories: [Study, CS]
tags: [Multi-programming, Multi-Tasking, Multi-processsing, Multi-Threading]
layout: post
toc: true
---

<br>
저번 포스팅에 이어, 이번 포스팅 에서는 Multi-programming, Multi-Taskng, Multi-processing, Multi-Threding을
다뤄보도록 하겠다.

### 목차
1. [Multi-programming](#1-multi-programming)
2. [Multi-Tasking](#2-multi-tasking)
3. [Multi-processsing](#3-multi-processing)
4. [Multi-Threading](#4-multi-threading)


<br>
## 1. Multi-Programming
---

- **단일 프로세서**에서 **여러 프로그램**을 <u>메모리에 동시에 올려놓고 수행</u>하는 것을 뜻한다.
- **멀티 프로그래밍**에서는 하나의 프로그램이 I/O 작업을 하는 경우 유휴 시간으로 대기하는 것이 아닌, <u>다른 프로그램이 그 동안 CPU 및 기타 리소스를 활용</u>한다.

![Multi-Programming](/assets/img/study/cs/multi-programming.png)
*출처: https://www.scaler.com/topics/multiprogramming-operating-system/*
- I/O 작업이 이루어지고 있는 `A`<u>작업은 CPU를 활용하지 않는다.</u>
- 따라서, <u>CPU는</u> `B` <u>작업을 실행</u>시킨다.
- 그 동안 `C` 작업은 `B` 작업이 끝난 후, <u>CPU 시간을 활용할 수 있도록 대기 상태에서 머무른다.</u>

- 이처럼 `멀티 프로그래밍`은 CPU의 유휴 시간을 최소화 하여, **<u>CPU 사용을 극대화 </u>** 하는데 목적을 두고 있다.

<br>
## 2. Multi-Tasking
---

- 다수의 작업(Task)을 <u>운영체제 스케줄링에 의해 번갈아가면서 처리</u>하는 것을 뜻한다(`Process`, `Thread` 모두 작업의 단위가 될 수 있음).
- `시분할 시스템(Time Sharing System)`을 적용하여 **<u>프로세스의 응답 시간을 최소화</u>** 하는데 목적을 두고 있다.

> 시분할 시스템이란?  

- 한 작업이 `CPU` 시간을 오래 잡고 있는 것을 방지하고자, 각 작업을 일정 시간(`time quantum` or `time slice`) 동안 번갈아 가면서 실행하는 것을 뜻함

![Time Sharing System](/assets/img/study/cs/time-sharing-system.png)

- `멀티 프로그래밍`과 `멀티 태스킹`의 가장 큰 차이점은 `시분할 시스템(Time Sharing System)` 이다.
- `멀티 프로그래밍`은 `time slice`에 의한 `CPU` 스위칭이 없기 때문에(**I/O 스위칭만 발생**) 하나의 작업이 `CPU` 시간을 오래 잡고 있을 수 있다는 문제가 발생한다.
  - 이는 높은 `CPU` 이용률과 긴 `응답 시간`의 결과를 가져온다.

<br>
## 3. Multi-Processing
---

- 두 대 이상의 **프로세서**가 다수의 **프로세스**를 협력적으로 **<u>동시에 처리</u>**하는 것(`프로세서`는 `CPU`와 같은 개념으로 생각하면 된다)
- 각 `프로세서`는 다수의 `프로세스`를 처리하며, 각 `프로세스`는 다수의 프로세서에 의해 처리된다.
- **각 프로세서가 자원을 공유하면서 프로세스를 처리하기 때문에, 하나의 프로세서가 멈춰도 작업은 정지되지 않는다.**

![Multi-Processing](/assets/img/study/cs/multi-processing.png)
*출처: https://levelup.gitconnected.com/*

<br>
## 4. Multi-Threading
---

> 스레드(Thread)란 ?

- 한 `프로세스` 내에서 구분지어진 **실행 단위**

- <u>프로세서가 여러 개인 경우</u> `멀티 스레딩` 을 통해 `병렬성(Parallelism)`을 높일 수 있다.
  - `프로세스`의 `스레드`들이 각각 다른 프로세서(`CPU`)에서 병렬적으로 수행될 수 있기 때문
- 만약, <u>프로세서가 한 개인 경우</u>에는 `멀티 스레딩`을 통해 `동시성(Concurrency)` 를 높일 수 있다.
  - 작업 중인 스레드가 `blocked(waiting)` 되더라도 다른 스레드로 스위칭이 가능하기 때문에, 실제로는 각 시간에는 한 작업만 수행되지만 병렬적으로 수행되는 것 처럼 보인다.

![Multi-Threading](/assets/img/study/cs/multi-threading.png)
*출처: https://levelup.gitconnected.com/*

## Reference
---
- [Definition of Multiprogramming Operating System](https://ecomputernotes.com/fundamental/disk-operating-system/multiprogramming-operating-system)
- [멀티 프로그래밍 & 시분할 시스템](https://velog.io/@profile_exe/%EC%9A%B4%EC%98%81%EC%B2%B4%EC%A0%9C-%EB%A9%80%ED%8B%B0-%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D-%EC%8B%9C%EB%B6%84%ED%95%A0-%EC%8B%9C%EC%8A%A4%ED%85%9C)
- [멀티 스레드](https://rebro.kr/174?category=504670)
  