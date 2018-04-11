---
title: Env 프리셋으로 Babel 설정 최적화하기
author: Hyeseong Kim
tags:
  - babel
  - javascript
references:
  - https://www.slideshare.net/ssuser2295821/babel-7
  - http://ahnheejong.name/articles/ecmascript-tc39/
---

> 주의사항: 이 글은 Babel 버전 7에서의 사용법을 기준으로 설명한다.

지난 번 리액트네이티브 모임에서 [Babel 7 소개 발표](https://www.slideshare.net/ssuser2295821/babel-7)를 했을 때, 슬라이드와 발표에서 아쉬움을 많이 느꼈습니다. 시간이 짧았던 것은 둘 째 치더라도 제 생각보다 바벨 릴리즈에 대한 관심도가 낮다고 느껴서  

# 바벨이란?
바벨은 자바스크립트 표준인 ECMAScript(이하 ES)의 최신 문법으로 작성된 코드를 실행할 수 있는 이전 버전 문법으로 변환해주는 트랜스파일러이다. (core-js 등의 주요 폴리필도 포함하고 있다.)

자바스크립트는 [ECMAScript 표준과, TC39 위원회](http://ahnheejong.name/articles/ecmascript-tc39/), 자바스크립트 커뮤니티를 통해 빠르게 발전해오고 있지만 브라우저들과 Node.js가 지원하는 자바스크립트는 이를 완전히 따라잡지 못하고 있다. 특히 구버전의 IE를 지원하거나 해야되는 상황도 많기 때문에 순수히 ES를 사용해 개발하는 것은 그림의 떡같은 상황이다.

> [ES 호환성 테이블](http://kangax.github.io/compat-table)에서 지원현황을 확인할 수 있다.

바벨을 사용하면 이런 호환성 걱정 없이 자바스크립트의 최신 문법을 자유롭게 사용할 수 있고, 이 후에 포함될 문법도 미리 사용해볼 수 있는 장점이 있다.

# 바벨 설정하기
이미 ES 문법을 사용하거나 리액트를 사용하는 프로젝트라면 이미 설정되어 있겠지만, 간단하게 설치하는 법과 설정하는 법을 살펴본 후 어떻게 **최적의 설정** 하는지 살펴보자.

# Env 프리셋 사용하기

# 모던 자바스크립트 사용하기