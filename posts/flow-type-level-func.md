---
title: Flow의 Type-level Application 소개
author: Hyeseong Kim
date: 2018-04-19
tags:
  - flow
  - javascript
  - typescript
---

> [타입 이론](https://en.wikipedia.org/wiki/Type_theory)에 대해 무지한 상태입니다.. 공부하면서 고치는데 무지하게 시간 쏟을 것 같으니 이상한 내용 있으면 서슴없이 알려주세요 :pray:

Flow의 고급기능에는 생각보다 잘 알려지지 않은 재밌는 것들이 많다.

미리 빌드되서 제공되는 유틸리티 함수들 중에서도 재밌는 게 많지만, 가장 재밌는 건 Type-level 함수로 직접 유틸리티를 만드는 일 같아 소개하는 글을 써본다.

# Type-level Function

Flow는 오직 Flow 서버에서 타입 추론 용도로만 사용될 타입정의을 지원하는데 이걸 [Type-level function이라고 부르는 것 같다.](https://github.com/facebook/flow/issues/30#issuecomment-346668903)(비공식)

```js
type __ReturnType = <T>(...args: any): T => T
```

엥? 이거 그냥 [제네릭](https://en.wikipedia.org/wiki/Generic_programming) 아닌가?

제네릭 표현식에 사용되는 타입 파라미터(`<T>`)가 사용되지만, 사용되는 위치가 약간 다르다.

다음은 제네릭과 함께 사용되는 예시이다.

```js
// Generic
type Either<L, R> = { type: 'Left', value: L } | { type: 'Right', value: R }

// Type-level Function
type EitherF = <L, R>(x: [L, R]) => Either<L, R>
type ArrayF = <A>(x: [A]) => Array<A>

// $Call
type ArrOfStrings = $Call<ArrayF, [string]>
type EitherOfStringNumber = $Call<EitherF, [string, number]>
```

제네릭의 경우 타입 파라미터가 선언(Declaration)에 들어가고, 타입수준 함수에서는 정의(Definition)에 들어가는 것을 볼 수 있다.

위 예제는 내가 이해하기 너무 어렵다. 앞서 정의해놓은 `__ReturnType` 통해 어떻게 쓰는지 알아보자.

# `$Call`

# `$ObjMap`, `$TupleMap`

# 사용예1. React Component의 PropType 추론하기

얼마전 트위터 타임라인에서 이런 글을 보게되었다.

> <blockquote class="twitter-tweet" data-lang="ko"><p lang="ko" dir="ltr">type PropsType&lt;T&gt; = T extends React.Component&lt;infer P&gt; ? P : {};<br>라이브러리에서 PropsType을 export 안하고 있지만 저걸로 뜯어냈다. (ex: PropsType&lt;YouTube&gt;[&#39;opts&#39;]) 타입스크립트 개쩜ㅋㅋㅋㅋㅋ flow는 이런거 못하지 ㅋㅋ</p>&mdash; ㄹ (@disjukr) <a href="https://twitter.com/disjukr/status/984633981973901312?ref_src=twsrc%5Etfw">2018년 4월 13일</a></blockquote>

TypeScript 2.8에서 Conditional Type이라는 기능과 함께 `infer` 키워드가 추가되면서, Flow처럼 리턴 타입을 추론해내거나 하는 일이 가능해졌다.

근데 저건 Flow로도 되지 않을까? (솔직히 ~~또~~ 안될까봐 불안했다) 일단,

![Challenge Aceepted](images/how-i-met-your-mother-challenge-accepted.jpg)

# 사용예2. Reducer 목록으로 부터 State 추론하기

# 사용예3. Object Nested-key 추론하기

# 마무리 하며

여느 Facebook 라이브러리들이 그렇듯, 문서에는 Concept과 Philosophy 위주로 써있고 MS처럼 진입장벽을 허물고 사용자를 포섭하려는 노력이 부족한 것 같다.

나날히 발전하는 TypeScript 진영을 보면서, 이미 종말의 치달은 Flow의 이슈트래커와 그래도 아직 Flow가 나은점이 있다고 위로하면서도 정작 자신의 프로젝트에는 TypeScript를 셋업하고 있는 나를 보면서 Flow 사용자로서의 자부심은 바닥을 치고 있는 요즘이다.