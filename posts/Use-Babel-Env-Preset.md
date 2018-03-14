---
title: Env 프리셋으로 Babel 설정 최적화하기
author: Hyeseong Kim
tags:
  - babel
  - javascript
---

> 주의사항: 이 글은 Babel 버전 7에서의 사용법을 기준으로 설명합니다.

지난 [Babel 7 소개 발표](https://www.slideshare.net/ssuser2295821/babel-7)를 준비하며 아직도 많은 프로젝트들이 `babel-preset-es2015`  + `babel-preset-stage-0` + `babel-polyfill` 조합을 사용하고 있다는 걸 알게됐다. 이렇게 설정하면 거의 모든 플러그인+폴리필들을 한꺼번에 포함하여 설정이 편리하기 때문이다.

간과하기 쉬운 부분은, Babel은 단순히 개발환경에서만 쓰는게 아니라 **트랜스파일된 코드가 런타임에 반영**된다는 점이다. 

구분없이 프리셋과 폴리필을 쓰면 실제로는 불필요한 코드가 포함되기 때문에 일일히 플러그인을 설정하거나 아예 [직접 프리셋 플러그인을 만들어 쓰는 경우](https://github.com/facebook/react-native/tree/master/babel-preset)도 있다.

바벨에선 최신 문법을 쓰기위해 필요한 플러그인과 폴리필을 한꺼번에 프리셋으로 제공하면서도, 실행환경에 불필요한 코드가 포함되는 문제점을 개선하기 위해 [Env 프리셋](https://babeljs.io/docs/plugins/preset-env)을 만들어 제공해왔고, Babel 7에서 또 대폭 개선될 예정이다.

# Env 프리셋이란?

`@babel/preset-env`는 [browserslist](https://github.com/ai/browserslist)의 표현식과 [ES6 compat table](https://kangax.github.io/compat-table/es6/) 데이터를 기반으로 지정한 실행환경에서 필요한 플러그인과 폴리필만 로드해주는 스마트한 프리셋이다. 그렇기 때문에 일단 바벨에서 제공되는 모든 플러그인들을 포함하고 있다.