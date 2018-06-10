---
title: 도커로 루비 온 레일즈 프로덕션 환경 구성하기
author: Hyeseong Kim
date: 2018-06-05
tags:
  - ruby
  - rails
  - docker
  - devops
references:
  - https://12factor.net
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

### 먼저 기반 이미지를 지정한다

```dockerfile
FROM ruby:2.5.1-alpine
```

프로그래밍 언어에서 제공하는 공식 런타임 이미지가 있다면 사용한다. 프로그래밍 언어는 Stable 버전이 있고 호환성에 상당히 민감하기 때문에 `latest`를 사용하지 않고 버전을 지정한다.

문제가 없다면 가벼운 alpine 기반의 이미지를 사용하는 편이 이미지를 경량화 하고 빌드시간을 최소화할 수 있다.

### Setup: 절차에서 필요한 의존성을 설치한다

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
    nodejs\<$NODE_VERSION \
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

### Build: 소스 코드 복사 후 빌드한다

루비의 경우 별도의 컴파일이 필요하지 않으므로, 소스코드를 복사하는 것만으로 실행 준비가 끝난다.

```dockerfile
WORKDIR /app

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

도커 이미지는 디렉티브마다 레이어를 만들고 빌드 할 때 이 레이어 단위로 캐시한다. 캐시 여부에 따라 빌드 시간이 대폭 차이나므로 레이어를 잘 나누어야 한다. 소스코드가 변경될 때 마다 패키지 설치부터 다시하면 매우 비효율적이기 때문에 `RUN` 디렉티브를 나누어주고 **변경이 적은 것부터 잦은 것 순 으로 배치**한다.

### Run: 실행 커맨드를 지정한다

```dockerfile
EXPOSE 3000
CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0", "-p", "3000"]
```

도커 컨테이너는 **[포트를 바인딩](https://12factor.net/port-binding)해서 사용**하므로 컨테이너 내부의 포트는 반드시 고정하고, 외부 서비스에 사용하는 포트는 `EXPOSE` 디렉티브로 명시한다.

`CMD` 디렉티브를 통해 컨테이너 커맨드를 지정하는데 데몬 형태가 아니라 **반드시 Foreground로 실행되는 커맨드여야 한다.**

### 추가: 볼륨 설정

도커 볼륨을 마운트해서 사용하게 될 데이터 경로들을 `VOLUME` 디렉티브로 명시한다.

```dockerfile
VOLUME ["/app/storage", "/app/log"]
```

## 애플리케이션 환경 설정

애플리케이션에서 자주 변경되는 [환경 설정은 환경 변수를 사용](https://12factor.net/config)하도록 설정해서 컨테이너로부터 쉽게 분리하고 컨테이너의 불변성을 유지할 수 있다.

레일즈에서는 설정파일에서 ERB 템플릿 지원해서 쉽게 환경변수를 이용할 수 있는데, 설정 템플릿을 지원하지 않는 프레임워크인 경우에는 템플릿 엔진을 이용해서 직접 구현해야 한다.

### DB 설정

레일즈의 ActiveRecord에서 사용하는 데이터베이스는 `config/database.yml` 파일에서 설정한다. 기본적으로는 SQLite3를 사용하도록 되어 있는데 프로덕션 모드에서 MariaDB를 사용하도록 `Gemfile`과 설정파일을 변경한다.

```diff
- gem 'sqlite3'

group :development, :test do
+  gem 'sqlite3'
end

group :production do
+  gem 'mysql2', '~> 0.5.1'
end
```

```yaml
default: &default
  encoding: utf8
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  adapter: sqlite3
  database: db/development.sqlite3

test:
  <<: *default
  adapter: sqlite3
  database: db/test.sqlite3

production:
  <<: *default
  adapter: mysql2
  host: <%= ENV.fetch('MYSQL_HOST') { 'db' } %>
  port: <%= ENV.fetch('MYSQL_PORT') { 3306 } %>
  username: <%= ENV['MYSQL_USER'] %>
  password: <%= ENV['MYSQL_PASSWORD'] %>
  database: <%= ENV['MYSQL_DATABASE'] %>
```

DB는 docker-compose 스택으로 함께 구성할 것이기 때문에 db:3306 으로 고정해도 무관하다. docker-compose 스택 외부의 DB를 사용할 가능성이 있으므로 일단 환경변수로 분리하되 기본 값을 지정한다.

### Logging 설정

레일즈는 프로덕션 모드 설정인 `config/environments/production.rb`를 보면 기본적으로 로깅을 `log/production.log` 파일에 하며 `RAILS_LOG_TO_STDOUT` 환경변수를 지정하면 stdout 스트림으로 로깅하도록 변경되게 끔 설정돼있다.

Docker와의 [연계를 위해서 stdout으로 로깅](https://12factor.net/logs)하도록 변경하고 반대로 파일 로그(+ 자동 로테이션) 옵션을 만든다.

```ruby
  stdout_logger = ActiveSupport::Logger.new(STDOUT)
  stdout_logger.formatter = config.log_formatter
  stdout_logger.level = ENV["LOG_LEVEL"] || :info
  stdout_logger = ActiveSupport::TaggedLogging.new(stdout_logger)
  config.logger = stdout_logger

  if ENV["ENABLE_FILE_LOG"] == "true"
    # 파일 로거를 생성한다. 자동으로 로테이션 되도록 설정할 수 있다.
    file_logger = ActiveSupport::Logger.new(config.paths["log"].first, 5, 10.megabytes)
    file_logger.formatter = config.log_formatter
    file_logger.level = ENV["FILE_LOG_LEVEL"] || :info
    file_logger = ActiveSupport::TaggedLogging.new(file_logger)
    # 로거를 교체하는 대신 로그 브로드캐스팅을 사용한다.
    config.logger.extend(ActiveSupport::Logger.broadcast(file_logger))
  end
```

### File Storage 설정

ActiveStorage 설정은 `config/storage.yml`에서 변경할 수 있는데 로컬 스토리지를 사용하는 경우는 변경할 부분이 특별히 없다. `storage/` 경로에 Docker Volume을 마운트해서 사용할 것이다.

## 이미지 빌드 & 컨테이너 실행

이미지를 빌드하기에 앞서 프로젝트 루트에 `.dockerignore` 파일을 추가해서 빌드 시 불필요한 컨텍스트 전송을 방지한다. 필요한 내용은 `.gitignore`와 동일하므로 복사해서 사용해도 된다.

```bash
cp .gitignore .dockerignore
```

Dockerfile이 있는 경로에서 `docker build` 커맨드로 이미지를 빌드할 수 있다. rails-docker라는 이름으로 이미지를 빌드한다.

```bash
docker build --tag rails-docker .
```

이미지가 성공적으로 빌드되면 `docker run` 커맨드로 컨테이너를 실행해본다.

로컬호스트에 MariaDB가 설치되어 있다면, 컨테이너 네트워크를 `host`로 지정해서 테스트해볼 수 있다. 테스트에 사용할 DB는 미리 생성해두자.

```bash
docker run -it -d \
    --net=host \
    -e MYSQL_HOST=localhost \
    -e MYSQL_USER=root \
    -e MYSQL_PASSWORD=password \
    -e MYSQL_DATABASE=rails-data \
    -e ENABLE_FILE_LOG=true \
    -v $(pwd)/data/storage:/app/storage \
    -v $(pwd)/data/log:/app/log \
    rails-docker
```

매번 환경 변수를 커맨드에 입력하는 대신 `.env` 파일을 사용할 수 있다. 이 경우 .env 파일이 git 레파지토리에 들어가지 않도록 `.gitignore`에 반드시 추가해주자.

```diff
# .gitignore
+*.env
```

나는 로컬호스트에 데이터베이스를 설치하는 것을 싫어해서 [MariaDB 컨테이너 이미지](https://hub.docker.com/_/mariadb)를 사용해서 테스트 했다. 같은 `.env` 파일을 컨테이너끼리 공유하도록 하면 쉽게 세팅할 수 있다.

```env
# MYSQL_HOST=db
# MYSQL_PORT=3306

MYSQL_USER=rails-user
MYSQL_PASSWORD=password
MYSQL_DATABASE=rails-data

# LOG_LEVEL=info

ENABLE_FILE_LOG=true
# FILE_LOG_LEVEL=info
```

```bash
docker run -it -d \
    --name rails-db \
    --env-file .env \
    -e MYSQL_RANDOM_ROOT_PASSWORD=yes \
    -v $(pwd)/data/db:/var/lib/mysql \
    mariadb:10.2

docker run -it -d \
    --env-file .env \
    --link rails-db:db \
    -v $(pwd)/data/storage:/app/storage \
    -v $(pwd)/data/log:/app/log \
    rails-docker
```

## ENTRYPOINT 스크립트

위 과정까지 하면 컨테이너 자체는 잘 실행 되지만 레일즈는 제대로 동작하지 않는다. 심지어 잘 안되는 이유가 하나만 있는 것도 아니다.

- DB가 구동되기 전에 레일즈가 먼저 실행된다. 접속할 DB가 없어서 레일즈도 덩달아 초기화에 실패한다.
- 레일즈가 처음 실행됐다면 `rails db:setup`을 통해 DB를 초기화해주어야 한다.
- 추가로 스키마의 변경이 있다면 `rails db:migrate`을 통해 마이그레이션해주어야 한다.
- 프로덕션 모드에서는 에셋 파이프라인의 라이브 컴파일이 비활성화되기 때문에 `rails assets:precompile`로 초기화해주어야 한다.
- 실행했던 컨테이너를 중단하고 재시작하면 안에 남아있는 pid 파일 때문에 실패한다.
  컨테이너의 라이프사이클과 적합하지 않은 임시 파일들을 삭제해주어야 한다.

이런 문제들은 레일즈가 도커랑 맞지 않아서 발생하는 것이 아니라, 대부분의 애플리케이션을 Dockerize 할 때 발생하는 공통적인 요구사항들이다.

`docker exec -it {CONTAINER_ID} sh`로 컨테이너 내부에 attach 해서 직접 실행해주면 문제를 해결할 수 있지만, **이건 아주 나쁜 방법**이다. `docker exec`를 사용하는 것은 대표적인 안티패턴에 해당한다.

도커 컨테이너는 Mortal해서 언제든지 삭제 될 수 있다고 전제해야 한다. `docker exec`를 통해 임의로 변경한 사항들은 컨테이너가 삭제되면서 같이 사라진다.  
(물론 `docker commit`을 통해 컨테이너 diff의 스냅샷을 보존할 수는 있지만 이렇게 사용하는 워크플로우는 흔하지 않다.)

컨테이너를 새로 생성할 때마다 직접 작업을 수행하는 것은 매우 비효율 적이므로 컨테이너의 진입점(Entrypoint)에서 자동화하는 것이 일반적이다.

> Alpine 이미지의 기본 진입점은 `/bin/sh`이다. 컨테이너는 엔트리 + 커맨드로써 실행되므로 rails-docker 컨테이너는 `/bin/sh bundle exec rails server ...`의 프로세스와 동일하다.

진입점은 Dockerfile의 `ENTRYPOINT` 디렉티브를 통해 변경할 수 있다.

```dockerfile
COPY docker-entry.sh .
RUN chmod +x ./docker-entry.sh
ENTRYPOINT ["./docker-entry.sh"]
```

프로젝트 루트에 `docker-entry.sh` 라는 쉘 스크립트를 추가해준다.

```bash
#!/bin/sh

# 임시 파일을 제거한다.
echo "Cleaning temp files..."
rm -rf tmp/*

# Asset이 초기화 됐는지 검사하고, 안되있으면 `rails asssets:precompile`로 초기화한다.
ASSETS_PATH="public/assets"
if [[ ! -n "$(ls -A $ASSETS_PATH 2>/dev/null)" ]]; then
    echo "Assets is not exist, precompiling assets..."
    bundle exec rails assets:precompile
fi

# netcat을 사용해서 DB가 준비될 때 까지 대기시킨다.
until nc -z $MYSQL_HOST $MYSQL_PORT; do
    echo "MySQL is not ready, sleeping..."
    sleep 5
done

# 현재 스키마 버전과 마지막 스키마 버전을 읽는다.

# `rails db:version` 명령어로 현재 DB에 셋업된 스키마 버전을 볼 수 있지만 불필요한 문자열이 포함되어 있으므로
# ActiveRecord::Migrator.current_version 을 대신 사용한다.
SCHEMA_VERSION="$(rails runner "puts ActiveRecord::Migrator.current_version" | tail -n 1)"
LAST_SCHEMA_VERSION="$(find "db/migrate" -name "*.rb" | xargs basename | sort | tail -n 1 | cut -d '_' -f 1)"

echo "Detected the current DB schema version is $SCHEMA_VERSION"
echo "Detected the last DB schema version is $LAST_SCHEMA_VERSION"

# DB가 아직 초기화 되지 않았다면 스키마 버전이 0이다.
# `rails db:setup`으로 초기화 해준다.
if [[ $SCHEMA_VERSION -eq "0" ]]; then
    echo "Initializing the database..."
    rails db:setup

# 현재 DB 스키마 버전과 최종 migration 파일의 버전을 비교하고
# 필요하면 `rails db:migrate`으로 마이그레이션 한다.
elif [[ $SCHEMA_VERSION -lt $LAST_SCHEMA_VERSION ]]; then
    echo "Updating the database..."
    rails db:migrate
fi

# 초기화 과정을 완료하면 컨테이너의 커맨드를 실행한다.
# `exec`를 사용하면 현재 프로세스에서 컨텍스트만 넘겨 사용할 수 있다.
exec "$@"
```

> 관례적으로 쉘 스크립트를 쓰지만, 사실 루비 런타임이 포함되어 있는 이미지기 때문에 진입점을 쉘 대신 루비로 작성해도 된다.
>
> 다른 대안으로 네트워크 대기, 환경변수 템플릿 등 일반적인 자동화 요구사항들의 구현을 제공하는 [Dockerize](https://github.com/jwilder/dockerize)라는 구현체도 있다.

다시 이미지를 빌드하고 실행해보면 자동적으로 필요한 과정들이 수행되고 성공적으로 레일즈 앱을 컨테이너로 실행할 수 있다.

# docker-compose로 서비스 스택 정의하기

[docker-compose](https://docs.docker.com/compose/overview)는 컨테이너 실행에 필요한 옵션들을 Yaml 형태의 DSL로 제공하고 여러 컨테이너들을 하나의 스택으로 관리할 수 있는 기능을 제공하는 도구이다.

DB, App, Web 서비스를 하나의 스택으로 정의하고 docker-compose가 제공하는 다양한 커맨드를 통해 쉽게 관리할 수 있다.

```yaml
version: '3.5'
services:
  db:
    image: mariadb:10.2
    env_file: .env
    environment:
      - MYSQL_RANDOM_ROOT_PASSWORD=yes
    volumes:
      - db-data:/var/lib/mysql
      - /etc/localtime:/etc/localtime:ro

  app:
    build: ./
    env_file: .env
    volumes:
      - app-assets:/app/public/assets
      - app-data:/app/storage
      - app-logs:/app/log
      - /etc/localtime:/etc/localtime:ro

  web:
    build:
      dockerfile: nginx.Dockerfile
    ports:
      - "80:80"
    volumes:
      - web-logs:/var/log/nginx
      - app-assets:/web/public/assets:ro
      - /etc/localtime:/etc/localtime:ro

volumes:
  db-data:
  app-data:
  app-assets:
  app-logs:
  web-logs:
```

```dockerfile
FROM nginx:alpine
COPY public /web/public
COPY nginx.conf /etc/nginx/conf.d/default.conf
VOLUME ["/var/log/nginx"]
```

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    sendfile on;

    gzip on;
    gzip_vary on;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    error_page 500 502 503 504 /500.html;
    error_page 404 /404.html;
    error_page 422 /422.html;

    location ~ .*\.(js|css|ico|txt|eot|ttf|woff|woff2)& {
        access_log off;
        log_not_found off;
    }

    location ~ ^/assets/ {
        root /web/public;
        gzip_static on;
    }

    location ~ ^/(500|404|422).html& {
        root /web/public;
    }

    location / {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://app:3000;
    }
}
```

# 마무리하며

SaaS급 정도되는 서비스 전체를 Dockerize하고 관리하는 데는 한 개 팀 정도는 있어야겠지만, 단일 호스트에서 실행하는 작은 In-house 서비스는 docker-compose 정도로도 충분히 관리할 수 있다.

또한 서비스 스택을 Dockerize하는 과정은 서비스 운영을 가능한 **최소한의 구성**으로 **자동화**하는 과정에 가깝기 때문에 애플리케이션이 요구하는 숨은 의존성과 운영에 필요한 공수를 미리 파악하는데 도움이 된다.

일단 지금 당장(레일즈를 처음 사용하며 스터디하는 데)은 별 쓸모 없는 야크쉐이빙이지만, 해놓으면 나중에 삽질할 일 조금이라도 줄겠지
