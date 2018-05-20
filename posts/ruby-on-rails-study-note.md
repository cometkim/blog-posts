---
title: 루비 온 레일즈 스터디 노트
author: Hyeseong Kim
tags:
  - ruby
  - rails
---

# Ruby on Rails 스터디 노트

이직한 회사에서 레일즈(Ruby on Rails)를 메인 프레임워크로 쓰게 됐다.

루비 자체도 생소하거니와 MVC 프레임워크 자체가 별로 익숙하지 않아 급하게 스터디 하면서 있는대로 기록함

## 스타일 가이드

루비와 레일즈 커뮤니티에서 사용하는 관례는 스타일 가이드를 참고한다.

- [루비 스타일 가이드 (한국어)](https://github.com/dalzony/ruby-style-guide/blob/master/README-koKR.md)
- [루비 온 레일즈 스타일 가이드 (한국어)](https://github.com/pureugong/rails-style-guide/blob/master/README-koKR.md)

## Gem

Ruby 패키지 관리자

## rbenv

Ruby 실행 환경 관리자

## Bundler

Ruby Gem 의존성 관리자

`Gemfile`, `Gemfile.lock` 을 통한 의존성 간 버전 추적/관리 제공

## [Rake](https://github.com/ruby/rake)

Ruby로 작성하는 Makefile

## [Rack](https://rack.github.io)

웹 커뮤니케이션을 위한 최소한의 인터페이스를 제공한다. 표준 응답코드, 헤더 등...

Handler - Middleware - Adopter 구성으로 웹 서버와 웹 프레임워크 사이의 인터페이스 역할을 해주어, 특정 프레임워크나 웹서버에 의존적이지 않은 다양한 미들웨어를 제공한다.

Rails, Sinatra 등 대표적인 루비 웹 프레임워크에 내장되어 있다.

레일즈 내에서의 사용은 [문서](https://guides.rorlab.org/rails_on_rack.html)를 참고

## Active Record

MVC의 M을 구성한다. 모델 계층을 구현하는 대표적인 패턴의 이름이자, 레일즈에서 제공되는 ORM 프레임워크의 이름

### Model 정의

`app/models` 경로에 `product.rb` 파일 생성

```rb
class Product < ApplicationRecord
    # ...
end
```

DB에 `product`(기본값으로 모델명의 snake_lower_case) 테이블과 자동으로 매핑된다.

### CRUD

- `Product.create` 메서드로 새로운 레코드 생성
- `Product.new` 메서드는 레코드 생성하지 않고 인스턴스만 반환
- `Product.save` 메서드로 인스턴스를 레코드와 동기화
- `Product.destroy` 메서드로 레코드 제거
- `Product.all` 모든 제품 레코드 조회
- `Product.first` 첫 번째 제품 레코드 조회
- `Product.find_by(field: value) 로 조건에 맞는 첫 번쨰 레코드 조회

더 많은 쿼리 인터페이스는 [문서](https://guides.rorlab.org/active_record_querying.html) 참고

### 레일즈 콘솔

`rails console` 명령어로 레일즈 콘솔을 실행한다.

실행되는 irb 세션을 통해 액티브 레코드를 조회할 수 있다.

## Action Controller

MVC에서 C를 구성한다. 레일즈의 컨트롤러 `ApplicationController`를 상속받는 루비 클래스이다.

### Controller 정의

`app/controllers` 경로에 `articles_controller.rb` 파일 생성 


```rb
class ArticlesController < ApplicationController
end
```

### Action

Router에서 지정한 액션명과 동일한 이름의 메서드를 정의한다.

```rb
Rails.application.routes.draw do
  # ...
  get 'articles/:id/edit', to: 'articles#edit'

end
```

```rb
class ArticlesController < ApplicationController
   def edit
     # ...
   end
end
```

### Params

## View

### eRuby (Embedded Ruby) & ERB

Ruby 내장 템플릿 시스템

ERB는 eRuby의 구현체로 레일즈에서 뷰를 구현하는 데 사용

- Expression: `<%= %>`
- Execution: `<% %>`
- Comments: `<%# %>`

### Partials

## JSON API 서버

Rails 5 부터 rails-api 통합, API-only 모드 지원

https://guides.rorlab.org/api_app.html

## 레일즈 제너레이터 사용

`rails generate` 명령어를 사용하면 모델, 뷰, 컨트롤러 코드를 직접 추가하지 않고 자동으로 생성할 수 있다.

### 모델

```
rails generate Article title:string description:text
```

DB 테이블 스키마 정의, DB 마이그레이션 코드, 모델 코드를 일괄적으로 생성한다.

스키마가 변경된 후면 반드시 `rails db:migrate`로 마이그레이션을 수행한다.

### 컨트롤러

```
rails generate controller Article index show new create edit update destroy
```

`ArticlesController` 컨트롤러와 지정한 액션들에 해당하는 메서드와, 뷰, 라우트 코드를 일괄적으로 생성한다.

### 스캐폴딩

리소스에 대한 모델, 뷰, 컨트롤러를 일괄적으로 생성하기 위해 `rails generate scaffold ...` 명령어를 사용한다.

```
rails generate scaffold Article title:string description:text
```

모델, 라우트, 컨트롤러, 일반적인 액션(`index`, `new`, `edit`, `show`, `create`, `update`, `destroy`)들과 해당 뷰까지 자동으로 생성한다.

레일즈의 기본적인 사용법은 여기서 생성되는 코드들만 따라가도 배울 수 있다.

## Webpacker

https://github.com/rails/webpakcer

레일즈의 에셋 파이프라인에 Webpack을 통합한다. 따라서 아래와 같은 것들이 가능해짐
- Webpack 을 통한 번들링
- Babel을 통한 ES6+ 사용
- React, Angular, Vue 등 뷰 프레임워크 사용
- TypeScript, CoffeeScript 등 pre-compile js 사용
- PostCSS 사용
- HMR 지원
