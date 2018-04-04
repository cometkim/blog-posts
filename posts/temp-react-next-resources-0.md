---
title: '임시글: React 16.3 릴리즈와 React Next 레퍼런스 모음'
author: Hyeseong Kim
date: 2018-04-04T00:00:00.000Z
tags:
  - react
---

# React 16.3 릴리즈

- [릴리즈 노트](https://github.com/facebook/react/releases)
- [공식 블로그](https://reactjs.org/blog/2018/03/29/react-v-16-3.html)
- [블로그 1](https://medium.com/@baphemot/whats-new-in-react-16-3-d2c9b7b6193b)

# 라이프 사이클 메서드 변경
추가
- `UNSAFE_componentWillMount`
- `UNSAFE_componentWillReceiveProps`
- `UNSAFE_componentWillUpdate`
- `static getDerivedStateFromProps`
- `getSnapshotBeforeUpdate`

제거 예정 (16.4부터 Deprecated, 17에서 삭제)
- `componentWillMount`
- `componentWillReceiveProps`
- `componentWillUpdate`

변경이유는 Async Rendering 관련 블로그에 설명됨

# New Context API
- [Context API로 Redux 대체? 1](https://medium.freecodecamp.org/replacing-redux-with-the-new-react-context-api-8f5d01a00e8c)
- [Context API로 Redux 대체? 2](http://blog.isquaredsoftware.com/2018/03/redux-not-dead-yet/)
- [Context API로 Redux 대체? 3](https://stackoverflow.com/questions/49568073/react-context-vs-react-redux-when-should-i-use-each-one)

내 요약은 이렇다.
- Context API 는 원래 있었다.
- 16.3부터 정식 API 릴리즈
- Context Provider를 제공하는 라이브러리들은 원래 대부분 변경 대비가 되어있다. (HOC 사용)
- 예를 들면 Redux의 `Provider`, `connect()`나 react-intl의 `IntlProvider`, styled-components의 `ThemeProvider` 등. 갖다 쓰는 사용자는 바꿀 것이 없다.

이제 Redux 안 쓰고 Context로 상태 전달 하면 되지 않아? 라는 흐름은 좀 이상하다. 이 아이디어를 차용해서 단일 스토어를 통한 상태관리 모델을 제공해주는게 Redux이고, 상태 전달만이 목적이라면 애초부터 Redux를 쓸 필요가 없다.

이번 16.3 릴리즈로 크게 변경된 라이브러리가 있다면 버리자. 관리가 제대로 안되고 있거나 앞으로도 그럴 가능성이 크다.

# New Ref API
- [공식 블로그](https://reactjs.org/docs/refs-and-the-dom.html)

# Async Rendering
- [`<AsyncMode>`(unstable_AsyncMode)](https://github.com/facebook/react/pull/12117)
- [공식 블로그](https://reactjs.org/blog/2018/03/27/update-on-async-rendering.html)
- [신선한? 레퍼런스 링크 모음](https://github.com/sw-yx/fresh-async-react)
- [또 다른 레퍼런스 링크 모음](https://github.com/koba04/react-fiber-resources)
- [JSConf 데모](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html) 

데모를 통해 리액트 Fiber에 또 다시 엄청난 성능/반응성 개선이 있는 것을 확인할 수 있다. 

이번에 크게 변경된 라이프사이클 메서드 등 모두 비동기 렌더링을 언급하고 있다. immer, Redux, MobX도 영향을 받을 것이라는 언급도 있는 듯하다(제대로 확인해볼 것) 지금 시점에서 가장 주목해야 할 변경사항이라는 확신이 든다.

> “Importantly, this is still the React you know. This is still the declarative component paradigm that you probably like about React.”

그리고 너무 반가워서 첨부한 인용, 내가 리액트를 쓰는 이유. 그러하다.

# 업그레이드 관련
- [`<StrictMode>`](https://reactjs.org/docs/strict-mode.html)
- [자동 스크립트](https://github.com/reactjs/react-codemod): 각종 레거시 패턴들을 자동으로 리팩토링해주는 스크립트가 있다. 이건 또 언제 만들어 놨지... 고맙게 ㅎㅎ

> Note that if you’re a React application developer, you don’t have to do anything about the legacy methods yet. The primary purpose of the upcoming version 16.3 release is to enable open source project maintainers to update their libraries in advance of any deprecation warnings. Those warnings will not be enabled until a future 16.x release.
> 

말그대로 16.3은 폭풍전야와 같은 릴리즈, 당장 사용자에게 다가올 변경은 미미하고 라이브러리 개발자를 위해 준비된 기간이라 할 수 있다. 여기서 관리 안되는 라이브러리 미리 걸러내지 못하면 다음 릴리즈부터 갈려나갈 것이다.

# 호환성 관련
하위 버전 지원을 버리지 않고 라이브러리를 업그레이드 하는 법
- https://github.com/reactjs/react-lifecycles-compat
- https://github.com/donavon/react-af

# 기타

리액트 팀이 직접 만든 외부 데이터 주입할 수 있는 라이브러리
- [create-subscription](https://github.com/facebook/react/tree/master/packages/create-subscription)
