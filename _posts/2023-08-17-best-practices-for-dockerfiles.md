---
title: "[Docker] Best practices for Dockerfile"
date: 2023-08-17
categories: [Infra, Docker]
tags: [Docker, Dockerfile]
layout: post
toc: true
---


## 목차
1. [Layer 최적화](#1-layer-수-최소화)
   1. [Docker 이미지 저장 방식](#11-docker-이미지가-저장되는-방식)
   2. [레이어 수 줄이기](#12-레이어layer-수-줄이기)
2. [명령문(Instructions) 정렬](#2-명령문instrucntions-정렬)
3. [패키지 최소화](#3-패키지-최소화)
   1. [애플리케이션 패키지 최소화](#31-애플리케이션-패키지-최소화)
   2. [OS 패키지 최소화](#32-os-패키지-최소화)
4. [Reference](#reference)

<br>

# 1. Layer 수 최소화
---

## 1.1 Docker 이미지가 저장되는 방식

Docker 이미지를 빌드하거나 다른 레포지토리로 부터 pull하는 경우 여러 hash 형태의 문자열이 화면에 보이는 것을 확인할 수 있다.

```bash
$ docker pull ubuntu:15.04

Using default tag: latest
latest: Pulling from library/ubuntu
c499e6d256d6: Already exists
74cda408e262: Pull complete
ffadbd415ab7: Pull complete
Digest: sha256:282530fcb7cd19f3848c7b611043f82ae4be3781cb00105a1d593d7e6286b596
Status: Downloaded newer image for ubuntu:15.04
docker.io/library/ubuntu:15.04
```

이렇게 분리된 데이터를 **레이어(Layer)** 라고 한다. 레이어는 Docker 이미지를 빌드할 때 Dockerfile에 정의된 명령문(Instructions)을 순서대로 실행하면서 만들어진다. 이 때, 이 레이어들은 각각 독립적으로 저장되고, 읽기 전용이기 때문에 임의로 수정할 수 없다.

```docker
# syntax=docker/dockerfile:1

FROM ubuntu:15.04
COPY . /app
RUN make /app
CMD python /app/app.py
```

각 명령은 하나의 레이어를 생성한다.

- `FROM` : ubuntu:15.04 Docker 이미지에서 레이어를 생성한다.
- `COPY` : Docker 클라이언트의 현재 디렉터리에서 파일을 추가한다.
- `RUN` : make 명령문을 통해 애플리케이션을 빌드한다.
- `CMD` : 컨테이너 내에서 실행할 명령을 지정한다.

Docker 이미지를 실행하고 컨테이너를 생성할 때 **기본 레이어(Image layers(R/O))** 위에 **컨테이너 레이어(Container layer(R/W))**라고도 하는 쓰기가 가능한 새 레이어를 추가한다. 즉, 아무리 많은 Docker 컨테이너를 실행하더라도 기존 읽기 전용 레이어(**Image layer**)는 변하지 않고, 컨테이너 마다 생성된 쓰기 가능 레이어(**Container layer**)에 데이터가 쌓이기 때문에 서로 겹치지 않는다.

> 새 파일 작성, 기존 파일 수정 및 파일 삭제와 같이 실행 중인 컨테이너에 대한 모든 변경 사항은 컨테이너 레이어에 기록된다.
{: .prompt-info}

![container-layers.png](/assets/img/infra/docker/container-layers.png){: width="500"}

이처럼 Docker 이미지 레이어는 이미지를 빌드할 때마다 캐시 되어 재사용 되기 때문에 빌드 시간 단축에 있어 중요한 역할을 한다. 단, 모든 명령문이 레이어가 되는 것은 아니다.

`RUN`, `ADD`, `COPY` 명령문이 레이어로 저장되고, `CMD`, `LABEL`, `ENV`, `EXPOSE` 등과 같이 메타 정보를 다루는 부분은 임시 레이어로 생성되지만 저장되지 않아 Docker 이미지 크기에 영향을 주지 않는다.

<br>

## 1.2 레이어(layer) 수 줄이기

Docker 이전 버전에서는 이미지 레이어 개수가 성능에 영향을 주었지만 현재는 그렇지 않다. 하지만, 레이어 수를 줄이는 것은 최적화 측면에서 도움이 될 수 있다.

레이어로 저장되는 `RUN`, `ADD`, `COPY` 명령문에서는 여러 개로 분리된 명령을 **체이닝(chaining)**으로 엮어 레이어 수를 줄일 수 있다.

```docker
# Before chaining (4 Layers)
RUN apt-get update
RUN apt-get -y install git
RUN apt-get -y install locales
RUN apt-get -y install gcc

# After chaining (1 Layer)
RUN apt-get update && apt-get install -y \
    gcc \
    git \
    locales
```

비록 레이어 개수가 적다고 Docker 이미지/컨테이너 성능에 영향을 주지 않지만 `Dockerfile` 의 가독성과 유지 보수 관점에서는 도움이 된다. 또한, 설치할 패키지를 알파벳 순으로 정렬한다면 가독성과 중복 방지 차원에서도 도움이 될 수 있다.

<br>

# 2. 명령문(Instrucntions) 정렬
---

위에서 언급한 것 처럼 Docker 이미지를 빌드할 때 Dockerfile에 정의된 명령문(Instructions)을 순서대로 실행한다. 따라서, 명령문을 잘 정렬하는 것 만으로도 Dockerfile을 최적화할 수 있다.

아래 `Dockerfile` 에서는 `COPY` 가 2번 실행된다. 첫 번째는 의존성 패키지가 명시된 파일, 두 번째는 애플리케이션 코드가 저장된 디렉터리다.

```docker
FROM python:3.8-slim-buster

WORKDIR /usr/src/app

COPY requirements.txt /usr/src/app
COPY django_project /usr/src/app

RUN pip install -r requirement.txt

CMD ["pip", "freeze"]
```

의존성 패키지의 경우 애플리케이션 코드에 비하여 자주 바뀌지 않기 때문에, 한 번 `COPY` 명령으로 생성된 레이어는 캐시되어 재사용 될 것이다. 하지만, 애플리케이션 코드의 경우 빌드할 때 마다 코드가 변경될 가능성이 높기 때문에 캐시가 자주 초기화 될 것이고, 이후 실행될 `RUN` 명령어에서 수행하는 의존성 패키지 설치 또한 매번 실행될 가능성이 높다. 이 경우에는 사용자가 의존성 파일(requirement.txt)에 버전을 명시하지 않거나(`*`) 또는 최신 상태(`latest`)로 명시한 경우 예기치 않은 패키지 버전의 변동이 발생할 수 있다.

> 따라서, 자주 변경되지 않는 `COPY` 명령은 애플리케이션 코드 복사 이전에 배치 하는 것이 Docker 이미지 빌드 시간을 단축하는데 유리하다.
{: .prompt-info}

<br>

# 3. 패키지 최소화
---
Docker 이미지의 크기를 최소화 하기 위해서는 불필요한 패키지를 설치하지 않거나 삭제 해야 한다.


## 3.1 애플리케이션 패키지 최소화
프로덕션 환경과 개발 환경의 패키지를 별도로 분리하여 프로덕션 환경에서는 개발용 패키지를 설치하지 않도록 설정한다.

**Pipfile**
```bash
[[source]]
url = "https://pypi.org/simple"
verify_ssl = true
name = "pypi"

[packages]
fastapi = "*"
websockets = "*"
celery = {extras = ["redis"], version = "*"}
uvicorn = "*"
asgiref = "*"

[dev-packages]
pytest = "*"

[requires]
python_version = "3.11"
python_full_version = "3.11.3"
```

프로덕션 환경에서는 `pipenv install` 개발 환경에서는 `--dev` 옵션을 사용하여 dev-pacakages 만 설치할 수 있다.

```bash
# 프로덕션 환경
pipenv install 

# 개발 환경
pipenv install --dev
```

<br>

## 3.2 OS 패키지 최소화

 우분투(Ubuntu)를 포함한 데비안(Debian) 계열의 리눅스에서 쓰이는 패키지 관리 명령어 도구인 `apt-get` 를 사용할 때, `--no-insatll-recommends` 옵션을 사용하면 추천 패키지를 설치 하지 않는다.

```docker
RUN apt-get update && \
    apt-get upgrade && \
    apt-get install -y --no-install-recommends netcat
```

또한, `apt-get` 을 사용해서 패키지를 설치하면 이 과정에서 생성된 캐시가 디렉토리에 저장된다. 따라서 `apt-get` 을 통해 패키지를 설치한 이후 해당 캐시를 삭제하는 것이 좋다.

```docker
RUN apt-get update && \
    apt-get upgrade && \
    apt-get install -y --no-install-recommends netcat && \
		rm -rf /var/lib/apt/lists/*
```


# Reference

---

- [https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#minimize-the-number-of-layers](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#minimize-the-number-of-layers)
- [https://jonnung.dev/docker/2020/04/08/optimizing-docker-images/#gsc.tab=0](https://jonnung.dev/docker/2020/04/08/optimizing-docker-images/#gsc.tab=0)
- [https://thearchivelog.dev/article/optimize-docker-image/](https://thearchivelog.dev/article/optimize-docker-image/)