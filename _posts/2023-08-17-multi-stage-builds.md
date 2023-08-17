---
title: "[Docker] Docker Multi-stage builds - (2/3)"
date: 2023-08-17
categories: [Infra, Docker]
tags: [Docker]
layout: post
toc: true
---

## 목차
1. [Before multi-stage](#1-before-multi-stage-buildsbuilder-pattern)
    1. [Builder pattern](#12-builder-pattern-example)
2. [Multi-stage builds](#2-multi-stage-builds)
    1. [Multi-stage build example](#21-multi-stage-builds-example)
    2. [Multi-statge 유용한 기능들](#22-multi-stage-유용한-기능들)
       1. [빌드 단계 이름 지정](#221-빌드-단계-이름-지정)
       2. [특정 빌드 단계에서 중지](#222-특정-빌드-단계에서-중지)
       3. [외부 이미지를 스테이지로 사용](#223-외부-이미지를-스테이지로-사용)
       4. [이전 스테이지를 새 스테이지로 사용](#224-이전-스테이지를-새-스테이지로-사용)
       5. [응용 예제](#225-특정-빌드-단계-중지--이전-스테이지를-새-스테이지로-사용ex-nextjs)
       6. [Buildkit 활용](#226-buildkit-활용)
3. [Reference](#reference)


<br>

# 1. Before multi-stage builds(Builder pattern)
---

`Multi-stage builds` 가 등장하기 전에는 `Dockerfile` 이미지의 크기를 줄이기 위하여 `Dockerfile`을 개발용으로 사용하고 다른 하나는 프로덕션 용으로 사용하는 것이 일반적 이었다. 이처럼 **두 개의 Docker 이미지**를 사용하여 하나는 빌드를 수행하고 다른 하나는 첫 번째 이미지에서 필요한 부분만 추출하여 재빌드 하는 것을 `Builder pattern` 이라고 한다.

### Builder pattern

- `Dockerfile.build`: 개발 및 빌드용, 애플리케이션을 구축하는 데 필요한 모든 것이 포함
- `Dockerfile`: 간소화된 프로덕션 빌드용, 애플리케이션과 이를 실행하는 데 필요한 종속 항목만 포함

## 1.2 Builder pattern example

- `Dockerfile.build`

```docker
# syntax=docker/dockerfile:1
FROM golang:1.16
WORKDIR /go/src/github.com/alexellis/href-counter/
COPY app.go ./
RUN go get -d -v golang.org/x/net/html \
  && CGO_ENABLED=0 go build -a -installsuffix cgo -o app .
```

- `Dockerfile`

```docker
# syntax=docker/dockerfile:1
FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY app ./
CMD ["./app"]
```

- `build.sh`
1. `Dockerfile.build` 를 빌드한다.
2. 아티팩트를 복사하기 위한 컨테이너를 생성한다
3. 간소화된 프로덕션용 이미지를 빌드한다.

```bash
#!/bin/sh
echo Building alexellis2/href-counter:build
docker build -t alexellis2/href-counter:build . -f Dockerfile.build

#아티팩트를 복사하기 위한 컨테이너 생성
docker container create --name extract alexellis2/href-counter:build
docker container cp extract:/go/src/github.com/alexellis/href-counter/app ./app
docker container rm -f extract

echo Building alexellis2/href-counter:latest
docker build --no-cache -t alexellis2/href-counter:latest .
rm ./app
```

> `builder pattern` 를 통해 프로덕션용 이미지를 만들기 위해서는 두 개의 Docker 이미지가 시스템에서 공간을 차지하며 로컬 디스크에도 아티팩트가 남게된다.
{: .prompt-info}

<br>

# 2. Multi-stage builds
---

`multi-stage build`는 별도의 파일을 분리하여 관리하는 번거로움 없이 `builder pattern`의 이점을 제공한다.

### muti-stage builds

- 다단계 빌드에서는 `Dockerfile` 내에서 `FROM` 구문을 여러번 추가해 사용하는 것으로, 마지막에 선언한 `FROM` 문이 최종 Base image로 사용함
- 이전 이미지에서 생성된 아티팩트를 복사히기 위해서는 `COPY --from=<base_image_number>` 를 사용하면 됨


> `multi-stage builds` 방식은 중간 이미지를 만들 필요가 없으며 로컬 시스템에 아티팩트를 추출할 필요 또한 없다.
{: .prompt-info}


## 2.1 Multi-stage builds example
---

- `Dockerfile`

```docker
# syntax=docker/dockerfile:1

FROM golang:1.16
WORKDIR /go/src/github.com/alexellis/href-counter/
RUN go get -d -v golang.org/x/net/html
COPY app.go ./
RUN CGO_ENABLED=0 go build -a -installsuffix cgo -o app .

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=0 /go/src/github.com/alexellis/href-counter/app ./
CMD ["./app"]
```

- `builder-pattern` 과 달리 복잡성이 크게 감소한 것을 확인할 수 있다.
- 기본적으로 스테이지에는 이름이 지정되지 않으며 **첫 번째 스테이지는 0부터 시작하여 정수로 스테이지를 참조**한다. 즉, `golang:1.16` 이미지가 가장 처음 스테이지 이므로 해당 스테이지의 번호는 `0` 번이다. 따라서, `alpine:latest` 에 있는 `COPY --from=0` 은 이전 단계에서 빌드된 아티팩트(`app`)를 현재 단계로 복사 하겠다는 뜻으로 해석할 수 있다.

> `COPY --from` 구문에서 선언하지 않은 아티팩트는 현재 이미지에 저장되지 않는다.
> <br> (Ex. Go SDK, intermediate artifacts)
{: .prompt-info}

<br>

## 2.2 Multi-stage 유용한 기능들
---

### 2.2.1 빌드 단계 이름 지정

```docker
# syntax=docker/dockerfile:1

FROM golang:1.16 AS builder
WORKDIR /go/src/github.com/alexellis/href-counter/
RUN go get -d -v golang.org/x/net/html
COPY app.go ./
RUN CGO_ENABLED=0 go build -a -installsuffix cgo -o app .

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /go/src/github.com/alexellis/href-counter/app ./
CMD ["./app"]
```

- 스테이지를 번호로 참조하는 대신, `AS <NAME>` 구문을 사용하여 각 스테이지를 이름으로 참조할 수 있다.


### 2.2.2 특정 빌드 단계에서 중지

```bash
docker build --target builder -t alexellis2/href-counter:latest .
```

- `Dockerfile` 빌드 시 `-target <STAGE NAME>` 옵션을 통해 **특정 스테이지 단계까지만 빌드하는 것이 가능하다.** 이는 다음과 같은 경우에 유용할 수 있다.
    - 특정 빌드 단계 디버깅
    - `debug` 스테이지 이미지를 빌드하여, 디버깅이 필요한 곳에서 재사용

### 2.2.3 외부 이미지를 스테이지로 사용

```docker
COPY --from=nginx:latest /etc/nginx/nginx.conf /nginx.conf
```

- `COPY --from=<base_image>`는 `Dockerfile` 내에 선언된 스테이지 뿐만 아니라, **로컬 이미지 또는 Docker 레지스트리에 보관되어 있는 이미지에서 복사 할 수 있다.**

### 2.2.4 이전 스테이지를 새 스테이지로 사용

```docker
# syntax=docker/dockerfile:1

FROM alpine:latest AS builder
RUN apk --no-cache add build-base

FROM builder AS build1
COPY source1.cpp source.cpp
RUN g++ -o /binary source.cpp

FROM builder AS build2
COPY source2.cpp source.cpp
RUN g++ -o /binary source.cpp
```

- `FROM` 구문을 사용하여 새로운 스테이지를 생성할 때, **이전에 사용했던 스테이지를 기본 이미지로 선택할 수 있다.**
- **특정 빌드 단계에서 중지**하는 옵션과 함께 사용한다면, **개발 환경과 프로덕션 환경을 분리하여 빌드 프로세스를 구성할 수 있다.**

### 2.2.5 특정 빌드 단계 중지 + 이전 스테이지를 새 스테이지로 사용(ex. Next.js)

```docker
# Build target base #
#####################
FROM node:14-alpine AS base
WORKDIR /app
ARG NODE_ENV=production
ENV PATH=/app/node_modules/.bin:$PATH \
    NODE_ENV="$NODE_ENV"
RUN apk --no-cache add curl
COPY package.json yarn.lock /app/
EXPOSE 3000

# Build target dependencies #
#############################
FROM base AS dependencies
# Install prod dependencies
RUN yarn install --production && \
    # Cache prod dependencies
    cp -R node_modules /prod_node_modules && \
    # Install dev dependencies
    yarn install --production=false

# Build target development #
############################
FROM dependencies AS development
COPY . /app
CMD [ "yarn", "dev" ]

# Build target builder #
########################
FROM base AS builder
COPY --from=dependencies /app/node_modules /app/node_modules
COPY . /app
RUN yarn build && \
    rm -rf node_modules

# Build target production #
###########################
FROM base AS production
COPY --from=builder /app/public /app/public
COPY --from=builder /app/.next /app/.next
COPY --from=dependencies /prod_node_modules /app/node_modules
CMD [ "yarn", "start" ]
```

- 개발 환경(`development`)은 **별도의 빌드 과정을 필요로 하지 않기** 때문에 **Base 이미지**(`base`)와 **패키지를 설치**(`dependencies`)하는 단계만 포함하면 된다. 따라서, 다음 명령어를 통해 개발 환경용 이미지를 빌드할 수 있다.`docker build --target development -t bulidtest/development:latest .`
- 운영 환경(`production`)은 **Base 이미지**(`base`)와 **패키지 설치**(`dependencies`) 그리고 **빌드(**`builder`**) 과정** 모두가 필요하다. 따라서, 다음과 같은 명령어를 통해 운영 환경동 이미지를 빌드할 수 있다.`docker build --target production -t buildtest/production:latest .`

### 2.2.6 BuildKit 활용

```bash
DOCKER_BUILDKIT=1 docker build .
```

- `BuildKit` 은 버전 23.0 부터 Docker Desktop 및 Docker Engine 사용자를 위한 기본 빌더이다. `BuildKit`은 다음과 같은 기능을 지원한다.
    - **사용하지 않는 빌드 단계 실행 감지 및 건너뛰기**
    <br>(Docker 레거시 버전은 `BuildKit`을 사용해야 `--target` 옵션 사용할 수 있다)
        
    - **독립적인 빌드 단계 구축 병렬화**
    <br>(스테이지가 상호 독립적인 경우 각 스테이지를 병렬로 빌드할 수 있다)
        

# Reference
---

- [https://docs.docker.com/build/building/multi-stage/#stop-at-a-specific-build-stage](https://docs.docker.com/build/building/multi-stage/#stop-at-a-specific-build-stage)
- [https://blog.alexellis.io/mutli-stage-docker-builds/](https://blog.alexellis.io/mutli-stage-docker-builds/)