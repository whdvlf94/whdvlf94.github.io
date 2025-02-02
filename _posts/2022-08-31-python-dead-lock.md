---
title: "[FastAPI] Session deadlock issue"
date: 2022-08-31
categories: [Programming, Python]
tags: [FastAPI, Session, deadlock, SQLAlchemy]
layout: post
toc: true
---

<br>
`production` 배포 전 동시성 테스트 단계에서 백엔드 서버가 request를 처리하지 못하고, 교착 상태(`deadlock`)로 멈추고 아래와 같은 에러를 뱉는 이슈가 지속적으로 발생했다.
> **교착 상태(deadlock)란?**  
> 두 개 이상의 작업이 서로 상대방의 작업이 끝나기만을 기다려, 결과적으로 아무것도 완료되지 못하는 상태를 가리킴

```shell
sqlalchemy.exc.TimeoutError: QueuePool limit of size 5 overflow 10 reached
```
<br>
결론부터 말하자면, 이번 issue는 `FastAPI`가 외부 Thread Pool(**ThreadPoolExecutor**)을 제어하는데 기반으로 하는 `starlette`의 `anyio worker thread`와 `SQLAlchemy`의  `session.execute` *blocking call*로 인해 발생하였다.

<br>
이번 포스팅에서는 `anyio worker thread`와 `session.execute`가 **무엇**이고, **왜 이러한 이슈가 발생**했는지 그리고 **어떻게 해결 했는지**에 대하여 작성해 보겠다.

## 목차
1. [교착 상태, 왜 발생했는가?](#1-교착-상태-왜-발생했는가)
2. [해결 방안](#2-해결-방안)
   1. [Thread, session engine 활용](#1-anyio-worker-thread-갯수와-connection-pool-크기-확장)
   2. [Native Corutine 활용](#2-path-operation-function과-dependency-function을-모두-async로-선언하여-native-corutine으로-동작시키기)
   3. [의존성 제거](#3-의존성-주입depends을-제거하고--session-객체를-직접-호출하기)
3. [문제 톺아보기](#문제-톺아보기)
4. [References](#references)


## 1. 교착 상태, 왜 발생했는가?
---
FastAPI 공식 Github issue에서 문제에 대한 다양한 의견들을 찾아보았고, 그 중 해당 issue의 본질적인 원인에 대하여 구체적으로 잘 설명한 답변이 있어 이를 참조하였다.

간단한 코드를 통해 설명하겠다.

  ```python
    import time
    import uvicorn

    from fastapi import Depends, FastAPI, Request
    from sqlalchemy import create_engine
    from sqlalchemy.orm import Session, sessionmaker
    from sqlalchemy.pool import QueuePool
    from anyio.lowlevel import RunVar
    from anyio import CapacityLimiter

    # SQLAlchemy setup
    engine = create_engine(
        'sqlite:///test.db',
        connect_args={'check_same_thread': False},
        poolclass=QueuePool,
        pool_size=4,
        max_overflow=0,
        pool_timeout=None,  # Wait forever for a connection
    )
    SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

    # FastAPI
    app = FastAPI()

    def get_db(request: Request):
        db = SessionLocal()
        try:
            yield db
        finally:
            db.close()

    @app.get('/')
    def index(db: Session = Depends(get_db)):
        # Some blocking work
        _ = db.execute('select 1')
        time.sleep(1)
        return {'hello': 'world'}

    # Run
    if __name__ == '__main__':
        uvicorn.run('app:app', reload=True, host='0.0.0.0', port=80)
  ```

  > `jmeter` 또는 `locust`를 사용하여, **thread pool size 이상의 requests**가 요청 되었다고 가정     

- 모든 요청이 들어왔을 때, 요청들은 의존성 주입(`Depends`)을 통해 세션 객체를 생성(`SessionLocal()`)하고, `yield` 키워드를 사용한다.
- 이 후 *path operation function*에서 실행되는 `db.execute` 는 *blocking operation*으로 [SQLAlchemy docs](https://docs.sqlalchemy.org/en/14/orm/session_basics.html)에 따르면 **세션은 쿼리가 실행되는 시점에서 *connection pool*에서 연결을 요청한다.**   


  > 위 예제에서는 SQLAlchemy *connection pool* 크기를 4로 설정했기 때문에, 4개 요청 이후에 실행된 `db.execute` 요청들은 연결되기를 기다리는 상태에 머무른다(`blocking`)
  {: .prompt-danger}
    ![pool_status](/assets/img/programming/python/sqlalchemy_pool_status.png)
    *4 개의 요청이 모두 checked out 된 상태*

<br>
FastAPI에서는 사용자가 *path operation function*을 `def`로 선언할 경우, 직접 호출되지 않고 외부 스레드 풀에서 실행된다. 이 때, `anyio`의 스레드 풀 사용하며, `anyio` 스레드 풀의 경우 **40개의 *worker thread*가 기본값으로 설정**되어 있다.   

  > 스레드 풀의 갯수를 변경하고 싶은 경우 아래와 같이 변경할 수 있다.   

  ```python
    from fastapi import FastAPI
    from anyio.lowlevel import RunVar
    from anyio import CapacityLimiter

    app = FastAPI()

    @app.on_event("startup")
    def startup():
        # N : number of threads
        N = 50
        RunVar("_default_thread_limiter").set(CapacityLimiter(N))  
  ```

- session과 연결되어 있는 4개의 요청이 작업을 마치게 되면, *path operation function* response 이후 `get_db()`의  `finally` 블록이 실행된다.
- 이 때, FastAPI는 `finally`블록을 실행시킬 때  `__exit__` 메서드를 호출하여 세션을 정리한다.

  > 위 예제에서 `get_db()` 역시 `def`로 선언되어 있기 때문에, ***path operation function* response 이후 코루틴이 아닌 외부 스레드 풀에서 실행**된다. 하지만, 모든 `anyio worker thread`는 SQLAlchemy 연결을 기다리는 상태에 놓여있기 때문에 **4개의 요청을 *connection pool*로 반납하지 못하는 교착 상태**에 놓이게 된다.
  {: .prompt-danger}

<br>
> 즉, **위 예제 코드의 근본적인 문제는 `anyio worker thread` 사용과 관련이 있다**는 것을 알 수 있다.
{: .prompt-info} 
  
<br>
## 2. 해결 방안
---

위 문제를 해결할 수 있는 방법으로는 

### 1. `anyio worker thread` 갯수와 `connection pool` 크기 확장    

  ```python
    #number of threads = 100
    RunVar("_default_thread_limiter").set(CapacityLimiter(100))

    #connection pool size = 100
    engine = create_engine(
        'sqlite:///test.db',
        connect_args={'check_same_thread': False},
        poolclass=QueuePool,
        pool_size=25,
        max_overflow=75,
        pool_timeout=None,  # Wait forever for a connection
    )
  ```   

  - `Thread`갯수를 늘리거나 `engine` 생성 시 `pool_size`의 크기와 `pool_timeout` 시간을 조절하여 문제를 해결할 수도 있다.
  - 하지만, 이 방법은 **서버의 리소스를 낭비**할 수 있으며, `pool_timeout`(connection 지속 시간)으로 인하여 **데이터가 제대로 처리되지 않고 유실될 수 있다는 큰 문제점**을 가지고 있다. 
  - 따라서, **이 방식은 채택하지 않았다.**


### 2. *path operation function*과 *dependency function*을 모두 `async`로 선언하여 `native corutine`으로 동작시키기   
  ```python
    async def get_db(request: Request):
    async def index(db: Session = Depends(get_db)):
  ```
  - MySQL 드라이버로 **`mysql+pymysql`(동기식)을 사용하고 있기 때문에 채택하지 않았다**  

    > MySQL 드라이버가 비동기를 지원하지 않더라도 `async`로 함수를 선언하여 사용할 수는 있다. 하지만, 퍼포먼스 측면에서 보았을 때, **`async` + `sync` 조합은 교착 상태 발생의 원인이 되므로 권장하지 않는 방법**이다.<br>
    [Sync/Async Function 이슈](https://github.com/tiangolo/fastapi/issues/4988#issuecomment-1156115189)
    {: .prompt-danger}   

<br>

### 3. 의존성 주입(`Depends`)을 제거하고  `Session` 객체를 직접 호출하기   
  - 교착 상태를 해결할 수 있는 다양한 해결 방법을 찾던 중, [원티드 FastAPI boilerplate](https://github.com/whdvlf94/fastapi-boilerplate)를 발견하게 되었고, Github 소스 코드를 통해 아이디어를 얻을 수 있었다.

- 해결 방법은 이렇다. ***path operation function*에서 의존성 주입(`Depends`)를 통해 세션 객체를 얻어왔던 기존 방식**에서 **실제 DB에 CRUD 작업이 이루어지는 곳에서 세션 객체를 다루는 방식으로 변경**하였다.
  ```python

    ##### Server #####
    # path : fastapi-boilerplate/app/serverpy
    from fastapi import FastAPI

    app = FastAPI()

    ##### Session #####
    # path : fastapi-boilerplate/core/db/session.py
    from sqlalchemy import create_engine
    from sqlalchemy.orm import scoped_session, sessionmaker, Session
    
    engine = create_engine(config.DB_URL, pool_recycle=3600)
    session: Union[Session, scoped_session] = scoped_session(
        sessionmaker(autocommit=True, autoflush=False, bind=engine),
        scopefunc=get_session_id,
    )     
      
    ##### Service #####
    # path :  fastapi-boilerplate/app/services/user.py
    from core.db import session #세션을 직접 호출하여 사용

    async def get_user_list(self, limit: int, prev: Optional[int]) -> List[User]:
        query = session.query(User)

        if prev:
            query = query.filter(User.id < prev)

        if limit > 10:
            limit = 10

  ```
  - 현재 개발 중인 서비스 특성상 **외부 Third party API와 통신 작업**과 **데이터 파싱 작업**이 많아 응답까지의 소요 시간이 큰 편이다.
  - 즉, 이러한 상황에서 의존성 주입(`Depends`)은 아주 큰 독이 될 수 있다.  

  > 따라서, 의존성 주입(`Depends`)을 제거하고, **DB에 CRUD 작업이 이루어지는 곳에서 세션 객체를 직접 호출하는 방식을 채택**하였다.
  {: .prompt-tip}

<br>

## 문제 톺아보기
---
사실 아직까지도 FastAPI GitHub issue 페이지에서 해당 이슈에 대해서 많은 사람들의 의견을 시간 날 때 마다 보고있다. 글을 곱씹을수록 어제까지만 하더라도 잘 이해되지 않던 개념이 이해가 되고, 조금 더 나은 방식으로 수정 할 수 있겠다 라는 생각이 들어 글을 자꾸 읽게 되는 것 같다. 

그래서 '<u>과연 이 해결 방안이 최선인가?</u>' 라는 질문을 해보았을 때, '<u>아직까지 내가 아는 한..내일은 또 모르지</u>'라고 답하고 싶다. 

<br>

## References
---
- [Dependency injection deadlock issue](https://github.com/tiangolo/fastapi/issues/3205)
- [FastAPI 속도 비교](https://github.com/tiangolo/fastapi/issues/2690)
- [FastAPI에서 SQLAlchemy session 다루는 방법](https://medium.com/wantedjobs/fastapi%EC%97%90%EC%84%9C-sqlalchemy-session-%EB%8B%A4%EB%A3%A8%EB%8A%94-%EB%B0%A9%EB%B2%95-118150b87efa)
- [Wanted FastAPI boilerplate](https://github.com/whdvlf94/fastapi-boilerplate)