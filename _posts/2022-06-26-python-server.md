---
title: "[Python] WSGI, ASGI"
date: 2022-06-26
categories: [Programming, Python]
tags: [WSGI, ASGI, Server, Python]
layout: post
toc: true
---


**<u>Python 환경에서 웹 서버와 Python 애플리케이션 간의 통신이 어떻게 이루어지는가</u>**에 대한 궁금증이 생기던 와중 잘 정리되어 있는 블로그가 있어 이번 글을 작성하게 되었다.
 
Python에는 Spring 계열의 Tomcat과 같은 WAS가 별도로 존재하지 않기 때문에 웹 서버와 애플리케이션 간의 통신을 위한 서버가 필요하다. 즉, 웹 서버와 애플리케이션 간의 인터페이스 역할을 해주는 미들웨어가 필요하다.

### 목차

1. [WSGI](#1-wsgi)
2. [ASGI](#2-asgi)
3. [WSGI, ASGI](#3-wsgi-asgi)

## 1. WSGI
> Web Server Gateway Interface  

<br>
WSGI 서버는 웹 서버가 **동적 페이지 요청**을 처리하기 위해 호출하는 서버이며, 일반적으로 `gunicorn` 과 `uwsgi` 를 가장 많이 사용한다.  

WSGI 서버는 **<u>웹 서버와 WSGI 애플리케이션 중간에 위치하고 있다.</u>** 웹 서버에 동적 요청이 발생하면 웹 서버가 WSGI 서버를 호출하고, WSGI 서버는 Python 프로그램을 호출하여 동적 페이지 요청을 대신 처리한다.
> WSGI 서버는 WSGI 미들웨어 또는 WSGI 컨테이너라고도 한다.  

<br>
웹 서버로부터 발생한 동적 요청은 결국 WSGI 애플리케이션이 처리하게 된다. 대표적인 WSGI 애플리케이션으로는 `Django`, `Flask`, `Tornado`가 있다.

### 1.1 WSGI 순서도

![wsgi-process](/assets/img/programming/python/wsgi_process.png)

## 2. ASGI
> Async Server Gateway Interface  

<br>
ASGI도 WSGI와 동일하게 웹 서버 및 애플리케이션 간의 표준 인터페이스를 제공하기 위한 서버이며, 일반적으로 `starlette`, `uvicorn` 을 사용한다.
> uvicorn은 `uvloop`와 `httptools`를 이용하여 ASGI를 구현한 서버이다. `uvloop`는 NodeJS V8 엔진에서 사용하는 libuv를 기반으로 작성되어 Node.js와 같은 비동기 처리 속도를 어느 정도 누를 수 있다는 장점이 있다.  

<br>  

다음과 같은 **<u>WSGI의 한계</u>**로 인하여 ASGI가 등장하게 되었다.
- `wsgi.websocket`을 사용할 수 있으나 표준화되지 않았다.
- HTTP/2(Concurrency) 동시성을 적용할 수 없다.
- 기본적으로 요청을 비동기로 처리하지 못한다.

대표적인 ASGI 애플리케이션으로는 `Django`, `FastAPI`, `Falcon(3.0 이상)`가 있다.

### 2.1 ASGI 순서도
![asgi-process](/assets/img/programming/python/asgi_process.png)


## 3. WSGI, ASGI
> WSGI와 ASGI 동시에 사용하기

uvicorn(ASGI)는 단일 프로세스로 비동기 처리가 가능하지만, 많은 양의 요청이 들어오는 경우 결국 **단일 프로세스라는 한계점**이 존재한다.  

gunicorn은 WSGI이자 프로세스 관리자 역할을 수행한다. 따라서, gunicorn을 `master` 프로세스로 설정하고, uvicorn을 멀티 프로세스로 띄워 각각의 프로세스를 `worker` 프로세스로 설정할 수 있다.  

그러나, `Kubernetes`, `Docker Swarm` 등 여러 시스템의 분산 컨테이너를 관리하는 클러스터가 있는 경우 uvicorn 단일 프로세스로 실행하고, `pod` 혹은 `container`의 replication을 늘리는 것이 더 효율적일 수 있다.([FastAPI docs - Replication 참조](https://fastapi.tiangolo.com/deployment/docker/#replication-number-of-processes))  


### 3.1 WSGI, ASGI 순서도
![wsgi-asgi-process](/assets/img/programming/python/wsgi-asgi-process.png)  

<br>

## Reference
[REST API 개발로 알아보는 WSGI, ASGI](https://blog.neonkid.xyz/249)  
[ASGI 웹 프레임워크 FastAPI 를 시작하며](https://breezymind.com/start-asgi-framework/)  
[FastAPI 톺아보기 - 부제: python 백엔드 봄은 온다](https://jybaek.tistory.com/890)