---
title: 도커로 루비 온 레일즈 프로덕션 환경 구성하기
author: Hyeseong Kim
date: 2018-06-05
tags:
  - ruby
  - rails
  - docker
  - devops
---

이직한 회사에서 루비 온 레일즈를 처음 쓰게 되어 스터디겸 파일럿 프로젝트를 진행하고 있다.

실제로 사용할 서비스이기 때문에 배포와 운영 단계까지 고려하고 있는데, 프로젝트를 Dockerize하여 [docker-compose](https://docs.docker.com/compose)로 운영하는 방향으로 정했다.

Dockerfile 이미지를 만들고 docker-compose로 서비스를 정의하는 것은 레일즈 뿐 아니라 대부분의 웹 앱이 비슷한 구성을 따라가기 때문에 이번에 하고 있는 레일즈 서비스 구성 과정을 참고 삼아 남겨두려고 한다.

# 일반적인 웹 애플리케이션 구성

![일반적인 웹 애플리케이션 구성](images/configuration-of-web-application.png)

일반적으로(적어도 내가 봐온 것들 기준으로) 작은 규모의 웹 서비스는 위 그림같이 프록시, 앱(+런타임) 서버, 스토리지(File, DB, Cache 등)으로 구성된다.

대표적인 예시로, APM(Apach + PHP + MySQL) 또한 이 구성에서 크게 벗어나지 않는다. 다만, APM처럼 하나의 웹서버가 프록시와 앱 실행 역할을 둘 다 수행하고 있다면 두 개의 웹 서버로 분리해주는 것이 서비스 구성성(Composability) 확보에 도움이 된다.

여기서는 NGINX, Ruby on Rails, MariaDB의 단일 인스턴스로만 스택을 구성한다. 서비스 이름을 줄여서 각각 Web, App, DB 라고 별칭한다.

# App 이미지 만들기

## Dockerfile 이미지

웹 애플리케이션의 Dockerfile을 작성하는 것은 패턴이 있기 때문에 단계별로 나눠서 서술해본다

### 먼저 기반 이미지를 지정한다.

```dockerfile
FROM ruby:2.5.1-alpine
```

프로그래밍 언어에서 제공하는 공식 런타임 이미지가 있다면 사용한다. 프로그래밍 언어는 Stable 버전이 있고 호환성에 상당히 민감하기 때문에 `latest`를 사용하지 않고 버전을 지정한다. 

문제가 없다면 가벼운 alpine 기반의 이미지를 사용하는 편이 이미지를 경량화 하고 빌드시간을 최소화할 수 있다.

### Setup: 절차에서 필요한 의존성을 설치한다.

```dockerfile
ENV NODE_VERSION 8.11.2

RUN apk add --no-cache --update \
    ca-certificates \
    linux-headers \
    build-base \
    libxml2-dev \
    libxslt-dev \
    tzdata \
    mariadb-dev \
    nodejs \
    yarn

RUN gem install bundler \
    && bundler config --global frozen 1
```

이 패키지들이 레일즈 애플리케이션을 구동하는 데 필요한 최소한의 패키지들이다.

도커 컨테이너는 언제 몇 번을 빌드하고 실행하던 선언된 동작이 동일하게 수행되도록 불변성과 멱등성을 보장해야한다.  의존성 설치과정에서 버전 지정이 확실하지 않으면 설치되는 의존성 모듈의 버전에 따라 동작이 바뀔 여지가 있다. 버전 지정을 위한 몇 가지 규칙을 정해놓으면 좋다.

- 호환성이 상관없는(주로 단일 기능만 수행하는 모듈) 경우 마지막 안정(Stable) 버전을 설치한다.
- 버전에 따른 호환성 변경이 있는 경우, 버전을 지정해서 설치한다.
- 버전 지정의 경우 `ENV` 디렉티브를 이용해서 명시하고 참조가 가능하도록 한다.

현대적인 패키지 매니져들은 대부분 버전 잠금(Lock) 기능을 제공하니 활용하자.

추가적으로 번들러의 경우 `frozen` 옵션을 활성화하면 컨테이너 내부에서 패키지 버전이 임의로 변경되지 않도록 강제할 수 있다.

### Build: 소스 코드 복사 후 빌드한다.

루비의 경우 별도의 컴파일이 필요하지 않으므로, 소스코드를 복사하는 것만으로 실행 준비가 끝난다.

```dockerfile
WORKDIR /usr/src/app

ENV RAILS_ENV production

# Node 모듈 설치
COPY package.json yarn.lock ./
RUN yarn install --production

# Gem 모듈 설치
COPY Gemfile Gemfile.lock ./
RUN bundle install --without development test

# 레일즈 앱 전체 복사
COPY . .
```

`WORKDIR` 디렉티브를 사용하면 작업 디렉토리가 새로 생성되고 커맨드들이 해당 작업 디렉토리를 기준으로 실행된다.

도커 이미지는 디렉티브마다 레이어를 만들고 빌드 할 때 이 레이어 단위로 캐시한다. 캐시 여부에 따라 빌드 시간이 대폭 차이나므로 레이어를 잘 나누어야 한다. 소스코드가 변경될 때 마다 패키지 설치부터 다시하면 매우 비효율적이기 때문에 `RUN` 디렉티브를 나누어주고 변경이 적은 것부터 잦은 것 순 으로 배치한다.

### Run: 실행 커맨드를 지정한다.

```dockerfile
EXPOSE 3000
CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0", "-p", "3000"]
```

도커 컨테이너는 **[포트를 바인딩](https://12factor.net/port-binding)해서 사용**하므로 컨테이너 내부의 포트는 반드시 고정하고, 외부 서비스에 사용하는 포트는 `EXPOSE` 디렉티브로 명시한다.

## ENTRYPOINT 스크립트



# docker-compose로 서비스 스택 정의하기

## App 서비스

## DB 서비스

## Web 서비스

## 서비스 확장

# 마무리하며

서비스가 SaaS급 정도되면 Dockerize하고 관리하는 데 한 개 팀 정도는 있어야겠지만, 단일 호스트에서 실행되고 관리되는 작은 In-house 서비스는 docker-compose 정도로도 충분하다.

그리고 서비스 스택을 Dockerize하는 과정은 서비스 운영을 **최소한의 구성**으로 **자동화**하는 과정에 가깝기 때문에 애플리케이션이 요구하는 숨은 의존성과 운영에 필요한 공수를 미리 파악하는데 도움이 된다.

일단 지금 당장은 별 쓸모 없는 야크쉐이빙이지만, 해놓으면 나중에 삽질할 일 조금이라도 줄겠지
