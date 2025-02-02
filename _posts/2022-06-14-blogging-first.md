---
title: "[Jekyll] 블로그 포스팅하는 방법"
date: 2022-06-14
categories: [Blogging, Toturial]
tags: [Blog, jekyll, GitHub, Git]
layout: post
toc: true
---

글로 나를 표현하는 일은 정말로 어렵다. 노트 필기는 나름 잘하는 편이다. 학교를 다닐 때나, 일을 하면서 배웠던 것들은 대체로 잘 정리를 했다. 그러나, 이것들을 하나의 이야기로 풀어나가는 과정은 정말이지 너무나도 어렵다. 어릴 때 책을 많이 읽지 않아서 일까 아니면 글 쓰는 것에는 별다른 재능이 없는 것 일까. 어찌됐든 **지금부터라도 글 쓰는 연습을 하려고 한다**. 처음부터 너무 잘 쓰려고 하기보다는 우선 글 쓰는 것에 초점을 맞춰서..!
(이 글 쓰는 것 마저도 오래 걸렸다)

> 글 쓰는 연습도 하고, 내가 배운 것들을 공유하기 위해 첫 번째 포스팅은 Jekyll Chirpy 테마를 사용하여 블로그 만드는 글로써 시작하겠다!

### 목차

1. [Settings](#1-settings)
2. [Chirpy theme install](#2-chirpy-theme-install)
3. [Chirpy run server](#3-chirpy-run-server)
4. [Upload GitHub Pages](#4-upload-github-pages)
## 1. Settings

> Jekyll 사용을 위해서는 `ruby` 가 필수이다. 아래에 첨부된 링크를 따라서 필요한 로컬 환경을 설정하자

---

- [**ruby 설치 링크**](https://jekyllrb.com/docs/installation/)

- 링크에서는 `ruby-3.1.1` 을 설치하라고 하는데, 현재 릴리즈 버전은 `ruby-3.1.2` 이므로 해당 내용을 변경해서 설치를 진행했다.

- ※ 환경이 바뀔 때 마다 개발 환경을 새로 설치하는 것이 귀찮다면 [**jekyll-docker**](https://github.com/envygeeks/jekyll-docker/blob/master/README.md) 를 통해 바로 배포할 수도 있다.

## 2. Chirpy theme install

> 블로그를 만들기 위해 여러 테마를 찾아보았는데 `Chirpy` 테마에 있는 `Categoires`와 `Archives` 레이아웃이 보기에 가장 깔끔하고 좋았다.

---

- Chirpy 테마는 두 가지 방식으로 설치할 수 있다.

### 2.1 [**chirpy-starter**](https://github.com/cotes2020/chirpy-starter/generate)

본 포스팅에서는 **<u>두 번째 방식</u>**으로 설치를 진행했다. `chirpy-starter` 를 이용해서 설치하는 방식은 아주 쉬우나, 참조한 블로그에 의하면 커스터마이징 할 때 계속 문제가 발생한다고 한다(사실 무슨 차이인지는 잘 모르겠다 😉)

### 2.2 [**chirpy fork**](https://github.com/cotes2020/jekyll-theme-chirpy/fork)

## 3. Chirpy run server

---

#### 3.1 Creating a New Site

- `chirpy fork` 를 통해서 새로운 레퍼지토리를 생성한다. 이 때, 레퍼지토리의 이름은 `<GH_USERNAME>.github.io` 로 생성할 것

#### 3.2 Clone your Repository

```shell
$ git clone <github URL>
```

- `fork`를 통해 생성된 레퍼지토리에서 git clone을 받는다.

#### 3.3 Installing Dependencies

```shell
#chirpy 초기화 작업
$ tools/init.sh
...
[INFO] Initialization successful!

$ bundle

#bundle 명령어 실행 시 권한 관련 에러가 나는 경우
$ bundle config set --local path 'vendor/bundle'
$ bundle install
```

- chirpy 초기화 작업을 진행하면 아래와 같은 파일들이 삭제된다.
  - .travis.yml
  - _posts 폴더 하위의 파일들
  - docs 폴더들

#### 3.4 Running Local

```shell
#WSL 버전
$ bundle exec jekyll s

#Mac 버전
$ jekyll server
```

- 서버가 정상적으로 동작하면 `http://127.0.0.1:4000/` 로 서버가 실행되는 것을 확인할 수 있다.


## 4. Upload GitHub Pages
---

- `master` 브랜치에서 변경 사항들을 적용하여 remote 서버로 push
- GitHub 서버에 파일들을 업로드 한 후, 해당 프로젝트 Settings > Pages 클릭
- Branch를 `gh-pages`로 변경하면 GitHub Page 설정 완료!
- main, gh-pages 브랜치를 제외하고 모두 삭제