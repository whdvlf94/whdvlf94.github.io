---
title: "[Docker] Docker 이미지 최적화(with Node.js)"
date: 2023-08-17
categories: [Infra, Docker]
tags: [Docker]
layout: post
toc: true
---

## 목차
 1. [Default node image](#1-default-node-image)
 2. [Docker Hub options](#2-docker-hub-options-nodebuster-vs-nodebullseye)
 3. [Slimer images](#3-image-tag-for-slimmer-images)
 4. [Alpine images](#4-image-tag-for-alpine-images)
 5. [Distroless images](#5-distroless-docker-images-for-nodejs)
 6. [Conclusion](#6-conclusion)
 7. [Reference](#reference)

<br>

이미지 크기와 취약성은 CI/CD 파이프라인 및 보안 태세에 큰 영향을 미칠 수 있기에 Base 이미지를 선택하는 것은 최적화를 위해 필요하다.

`Node.js` 이미지 빌드 시 사용할 수 있는 옵션으로는 Core Node.js 팀에서 유지 관리하는 **공식 Node.js 이미지**, Docker Hub repository에서 제공하는 **tag가 달린 이미지**, Google에서 관리하는 **distroless 이미지** 등이 존재한다.

# 1 Default node image
> node

Node.js Docker 팀에서 공식적으로 관리하며 여러 Docker Base 이미지 태그가 포함되어 있다. 해당 태그는 다양한 OS 버전(Debian, Ubuntu, Alpine 등)과 Node.js 런타임 자체에 매핑되는 여러 Docker Base 이미지 태그를 포함한다. 또한, amd64 또는 arm64x8(Apple M1)과 같은 CPU 아키텍처를 대상으로 하는 특정 버전 태그도 존재한다

```docker
FROM node
```

위 이미지를 빌드하게 되면, `node:latest` 버전으로 설치가 진행되며, 이미지 크기는 1GB 이다. 해당 이미지에서 발생할 수 있는 종속성 및 보안 취약성은 다음과 같다.

- **총 409개의 종속성**: `curl/libcurl4`, `git/git-man` 또는 `imagemagick/imagemagick-6-common`과 같은 운영 체제 패키지 관리자를 사용하였을 때 409개의 오픈 소스 라이브러리가 감지되었음
- **총 289개의 보안 문제**: 위 종속성 내에서 Buffer Overflows, Use After Free errors, Out-of-bounds Write 등 총 289개의 보안 문제가 발견되었음
- Node.js 18.2.0 런타임 버전은 [DNS Rebinding](https://security.snyk.io/vuln/SNYK-UPSTREAM-NODE-2946423), [HTTP Request Smuggling](https://security.snyk.io/vuln/SNYK-UPSTREAM-NODE-2946427), [Configuration Hijacking](https://security.snyk.io/vuln/SNYK-UPSTREAM-NODE-2946717) 등 7 가지 보안 문제에 취약함
- 또한, 해당 이미지 내에는 wget, git, curl과 같은 유틸리티가 default 로 설치되어 있다

<br>

# 2 Docker Hub options node:buster vs node:bullseye
> node:buster, node:bullseye


Node.js Docker Hub 레포지토리에서 사용 가능한 태그를 찾아보면 `node:buster` , `node:bullseye`를 찾을 수 있다. 두 태그는 모두 Debian 배포 버전을 기반으로 하고 있으며, **buster** 태그는 24년에 종료되는 Debian 10에 매핑되므로 좋은 선택은 아니다. **bullseye** 태그는 26년 6월 경 종료되는 Debian 11(Current stable release)에 매핑되므로 bullseye 태그를 사용하는 것이 보다 더 적절하다([Debian LTS](https://wiki.debian.org/LTS))

```docker
FROM node:bullseye
```

하지만, 위 방식으로 이미지를 빌드하면 `node` 이미지와 완전하게 동일한 종속성 및 보안 취약성 결과가 나타난다. 이는 `node`, `node:buster`, `node:bullseye` 모두 동일한 Node.js 이미지 태그를 가리키기 때문이다.

<br>

# 3 Image tag for slimmer images
> node:bullseye-slim

slim이 포함된 이미지는 기본 태그에 포함된 공통 패키지를 포함하지 않으며 node를 실행하기 위한 최소한의 패키지만 포함한다.

```docker
FROM node:bullseye-slim
```

위 이미지 크기는 약 245MB 이며, 콘텐츠를 스캔하면 97개의 종속성과 56개의 취약점이 나타난다. 기본 `node` 이미지와 비교해 보았을 때 이미지 크기 및 보안 태세 측면에서 더 나은 것을 확인할 수 있다.

<br>

# 4 Image tag for alpine images
> node:alpine

alpine(Alpine Linux) 버전은 slim 버전 보다 더 작은 약 180MB 이하의 이미지 크기를 갖는다. 이는 소프트웨어 설치 공간이 더 작다는 것을 의미하고 이는 취약성으로 나타나는 부분이 더 적다는 것을 의미한다.

```docker
FROM node:alpine
```

위 이미지의 콘텐츠를 스캔하였을 때, 총 16개의 운영 체제 종속성과 2개의 보안 취약점이 감지 되었으며 이는 `slim` 보다 이미지 크기 및 보안 태세 측면 모두에서 더 좋은 선택 임을 알 수 있다.

<br>

**그렇다면** `alpine` **이 가장 최선의 선택인 걸까?**

이미지 크기 및 보안 태세 측면에서는 좋은 선택일 수 있다. 하지만, Alpine 프로젝트는 **musl을 C 표준 라이브러리의 구현으로 사용**하는 반면 `bullseye` **또는** `slim` **과 같은 Debian의 Node.js 이미지 태그는** `glibc` **구현에 의존**한다. 

이러한 차이는 기본 C 라이브러리 차이로 인한 성능 문제, 기능적 버그 또는 잠재적인 응용 프로그램 충돌을 야기할 수도 있다. 뿐만 아니라, Node.js Docker 팀은 Alpine을 기반으로 하는 컨테이너 이미지 빌드를 공식적으로 지원하지 않는다. **따라서, Alpine 기반 이미지 태그는 실험적이고 일관성이 부족할 수도 있게 때문에 운영(Production) 단계에서 사용하기에 적합하지 않다.**

> Node.js Unofficial Builds Project
> 
> 
> Unofficial-builds attempts to provide basic Node.js binaries for some platforms that are either not supported or only partially supported by Node.js. This project does not provide any guarantees and its results are not rigorously tested. Builds made available at [Node.js](http://nodejs.org/) have very high quality standards for code quality, support on the relevant platforms platforms and for timing and methods of delivery. Builds made available by unofficial-builds have minimal or no testing; the platforms may have no inclusion in the official Node.js test infrastructure. These builds are made available for the convenience of their user community but those communities are expected to assist in their maintenance.


<br>

# 5 Distroless Docker Images for Node.js
> gcr.io/distroless/nodejs:16

`distroless` 이미지는 애플리케이션과 해당 런타임 종속성만 대상으로 하기 때문에 `slim` 이미지 태그 보다 더 적은 용량을 차지한다. 즉, 컨테이너 패키지 매니저, 쉘 그리고 기타 범용 도구 종속성이 없으므로 작은 크기와 취약성을 띠고 있다.

```docker
FROM gcr.io/distroless/nodejs:16
```

<br>

Distroless 컨테이너 이미지에는 소프트웨어가 없기 때문에 Docker에서 Multi-stage 워크 플로우를 사용하여 컨테이너에 대한 종속성을 설치하고 이를 `distroless` 이미지에 복사하여 사용한다.

```docker
FROM node:16-bullseye-slim AS build
WORKDIR /usr/src/app
COPY . /usr/src/app
RUN npm install

FROM gcr.io/distroless/nodejs:16
COPY --from=build /usr/src/app /usr/src/app
WORKDIR /usr/src/app
CMD ["server.js"]
```

`distroless` 이미지는 Debian 기반으로 되어 있으며, 안정적인 릴리즈(current stable release) 버전을 이미지로 제공한다. 하지만, 세분화된 Node.js 런타임 버전을 제공하지 않고 자주 업데이트되는 범용 `nodejs:16` 태그를 사용하거나 특정 시점의 SHA256 해시를 기반으로 설치해야 한다.

<br>

# 6 Conclusion

위 다양한 경우를 토대로 보았을 때, 가장 이상적인 Node.js Docker 이미지는 **안정적(stable)**이고 **장기 지원 버전(active Long Term Support Version)**이 있어야 하고, **최신 Debian OS를 기반으로 하는 slim 버전**이다.

```docker
FROM node:16-bullseye-slim
```

<br>

# Reference
---

- [https://snyk.io/blog/choosing-the-best-node-js-docker-image/](https://snyk.io/blog/choosing-the-best-node-js-docker-image/)
- [https://thearchivelog.dev/article/optimize-docker-image/](https://thearchivelog.dev/article/optimize-docker-image/)
- [https://github.com/vercel/next.js/discussions/16995](https://github.com/vercel/next.js/discussions/16995)
- [https://blog.alexellis.io/mutli-stage-docker-builds/](https://blog.alexellis.io/mutli-stage-docker-builds/)