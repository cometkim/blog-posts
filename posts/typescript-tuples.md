---
title: TypeScript 튜플 타입 요리하기
author: Hyeseong Kim
date: 2020-08-30
tags:
  - typescript
---

지난 4월 말에 TypeScript Korea 그룹에서 [타입스크립트에게 내 의도를 이해시키는 방법](https://youtu.be/bfSKqscC8kc) 이라는 주제로 발표했었습니다.

그 때는 막상 발표를 정해놓고 주제를 잘 정리하지 못해서 두서없이 이런저런 얘기를 했었습니다. 대략 어떤 내용이였냐면

- 타입스크립트가 타입을 추론하는 방식
- 타입 추론이 안전한 코드 작성에 어떻게 도움이 되는지
- 타입 추론을 더 쉽게 사용하기 위한 Type-level 유틸리티를 만드는 방법 (feat. `infer`)
- 안전하지 않은 타입을 안전한(컴파일러가 추론 가능한) 타입으로 캐스팅 하는 방법
- 튜플 타입 다루는 법

이 중에서 튜플 타입을 다루는 방법들에 대해서는 나중에 꼭 글을 써야지라고 생각만 하고 어기적거리고 있다보니 어느새 5개월이 지나고, 어느새 [TypeScript 4.0이 출시](https://devblogs.microsoft.com/typescript/announcing-typescript-4-0/)되었으며, Variadic tuple이라는 새로운 문법의 등장으로 많은 변화가 생겼습니다.

상황이 이렇게 되니 한 편으로는 제 게으름에 대해 자괴감도 들고, 또 한 편으로는 새기능 소개를 한꺼번에 할 수 있으니 잘 됐다는 생각도 들고 그러네요.

이 글은 튜플 형태의 타입 정의 특성을 가볍게 다루기도 하지만 Conditional typing과 infer 키워드를 많이 활용하기 때문에 해당 기능에 대한 이해도가 어느정도 필요합니다.
(타입스트립트의 타입 시스템을 타의 추종이 불가능할 정도로 강력하게 만들어주는 고급 기능이자 제가 아직 TypeScript를 사용하는 유일한 이유이기도 합니다)

## 튜플의 특징

```typescript
type tuple = readonly [1, 'hello', false];
```

일반적으로 튜플은 불변 구조이기 때문에 `readonly` 키워드를 명시적으로 수식했습니다. 사실 어떤 인덱스에 어떤 자료가 들어있는지 기술한 시점부터 `readonly` 키워드가 명시적으로 있던지 없던지 그 성질은 동일합니다.

```typescript
// $ExpectType readonly ['sun', 'mon', 'tue', 'wed', 'thu', 'fri', 'sat']
const days = ['sun', 'mon', 'tue', 'wed', 'thu', 'fri', 'sat'] as const;
```

이렇게 `as const`를 기술하면 쉽게 값의 정의로부터도 튜플 타입을 가져올 수 있습니다.

이렇게 정의된 튜플 타입의 값은 몇가지 고유한 성질을 가집니다.

1. Array 인터페이스를 만족하는 객체이기도 합니다.
2. 인덱스 타입도 불변이며, 컴파일러가 그 정확한 값을 기억하고 있습니다.

즉, 위에서 정의한 `days` 의 타입은 이런 형태로도 기술될 수 있습니다.

```typescript
{
  0: 'sun',
  1: 'mon',
  2: 'tue',
  3: 'wed',
  4: 'thu',
  5: 'fri',
  6: 'sat',
  readonly length: 7,
}
```

타입스크립트로 범용적인 유틸리티를 만들기 위해서는 이런 특징을 잘 활용해야 합니다. 제일 일반적인 형태를 뽑아보자면 `readonly unknown[]` 같이 뭉개지겠지만, 적어도 0번 인덱스와 `length` 는 참조할 수 있다는 사실은 변하지 않습니다. 따라서, 다음과 같은 유틸리티를 만들 수 있습니다.

```typescript
type FirstEntry<T extends readonly unknown[]> = T[0];
type Length<T extends readonly unknown[]> = T['length'];

// 각 인덱스들의 가장 일반적인 형태(best common type)은 number 입니다.
// $ExpectType 'sun' | 'mon' | 'tue' | 'wed' | 'thu' | 'fri' | 'sat'
type Day = typeof days[number];

// $ExpectType 'sun'
type FirstDay = FirstEntry<typeof days>;

// $ExpectType 7
type CountOfDays = Length<typeof days>;
```

실제로 타입스크립트를 사용하면 Iterable한 값과 Enumerable한 타입 정의가 동시에 필요한 경우가 많이 있기 때문에 여기까지만 알아도 벌써 꽤 유용합니다.

하지만 아쉬운 점이 있다면, 인덱스가 항상 0으로부터 시작한다는 사실을 통해 `FirstEntry<T>` 유은 만들 수 있었지만 이에 대칭되는 `LastEntry<T>` 는 만들지 못했다는 사실이 대칭성을 사랑하는 저를 굉장히 불편하게 만들었습니다. "마지막 인덱스는 항상 `length` 값보다 1 작다" 라는 규칙을 알고 있지만 기존 타입시스템에서 표현할 방법을 모릅니다.

기본 타입추론의 기능의 한계를 만났기 때문에 `infer`가 등장할 차례입니다. `LastEntry<T>` 를 만드는 방법을 이어서 설명드리겠습니다.

## 함수 시그니쳐와 튜플

일반적인 함수 시그니쳐의 타입도 튜플과 연관이 있습니다.

```typescript
interface Callable {
  (...args: any[]) => any
}
```

바로 위와 같은 가장 일반적인 형태(best common type)의 함수 정의에서 rest parameter 부분이 튜플(여기서는 `any[]`)이거든요.

이런 rest parameters로 부터도 실제로 타입을 추론해낼 수 있는데, 타입스크립트에 내장되어 있는 유틸리티 타입 중 하나인 `Parameters<T> `의 정의가 이 성질을 활용합니다.

```typescript
type Parameters<T> = T extends (...args: infer U) => any ? U : never;
```

`Parameters<T>` 를 사용하면 함수의 파라미터 배열의 형태를 튜플 형태로 추론해낼 수 있습니다.

이 성질을 조금 응용하면, 우리는 아까 만난 문제를 쉽게 해결 할 수 있습니다.

```typescript
type DropFirst<Tuple extends readonly unknown[]> =
  ((...tail: Tuple) => any) extends ((head: unknown, ...tail: infer Tail) => any)
  ? Tail : never;

type Last<Tuple extends readonly unknown[]> = Tuple[DropFirst<Tuple>['length']];

// $ExpectType 'sat'
type LastDay = Last<typeof days>;
```

이해가 되시나요? 함수 시그니쳐에서 튜플을 추론할 수 있다는 성질을 통해 튜플 제약사항을 가진 타입 파라미터 요소(`head`)를 하나 잘라내었습니다.

Conditional typing에 익숙하지 않으시면 햇갈릴 수 있기 때문에 풀어서 설명해보겠습니다.

먼저, `infer`는 타입스크립트에게 타입 추론을 위한 특별한 규칙이 있음을 알릴 때 쓰이는 키워드로, 오직 Conditional type의 조건 정의 부분에서만 사용할 수 있습니다. 주어진 제약사항 맥락에서 `infer`를 만나면 타입스크립트는 "실제 타입"을 추적해서 조건이 참인 경우와 거짓인 경우를 각각 "브랜칭" 해주는 역할을 합니다.

타입 파라미터인 `Tuple`에서 일부분을 추론하기 위해, "항상 참인 제약사항"을 하나 만들었습니다. 타입 파라미터 `Tuple` 에 `typeof days` 를 전달했을 때, 제약사항 식은 `((...tail: typeof days) => any) extends ((head: unknown, ...tail: infer Tail) => any)` 이 되는데 여기서 `infer` 에 의해 추가 파라미터 `Tail`이 정의됩니다.

타입 파라미터 `Tuple`은 이미 "튜플"이라는 제약사항을 가지고 있어 제약사항은 어떤 경우라도 참이 되며, `head` 하나를 제외한 나머지가 타입스크립트가 추론하는 `Tail`이 됩니다. 익숙해지면 간단한 트릭입니다.

## 타입레벨 재귀와 순회

타입시스템에서는 `새로운 타입 = 기존 타입 * 기존 타입` 처럼 기존의 타입들의 조합을 통해 새로운 타입을 정의할 수 있습니다. 

당연히 새 타입을 정의할 때, 자신자신의 정의를 재활용 하는 것은 불가능 합니다.

```typescript
type A = A;
//   ~
//    ^_____ Type alias 'A' circularly references itself.(2456)
```

정의하는 대상이 모호해지기 때문에 이는 직관적으로 당연한 것으로 느껴집니다.

하지만 추상 자료형을 정의할 때면 자기 자신의 정의를 활용할 일이 생깁니다.  
예를 들면, 대표적인 자료형 중 하나인 단방향 연결 리스트를 타입으로 표현하면 이런식입니다.

```typescript
type List<T> = {
  data: T,
  next?: List<T>,
};
```

타입스크립트를 포함한 많은 타입시스템들은 이렇게 추상 자료형을 포함하기 위해 제한적으로 자기 자신의 정의를 참조하는 재귀적인 형태의 타입 정의를 허용하고 있습니다.

타입스크립트의 경우에는 바로 위 코드처럼 레코드 형태의 타입의 필드 타입에서 자신의 타입을 참조하는 것이 허용됩니다.  `list.next?.next?.next?.next?.next` 같은 값의 타입이 결정적으로 추론될 수 있는 유일한 구조입니다.

아까 튜플의 부분을 조작하기 위해서 함수 시그니쳐를 사용했 듯이, 튜플 타입을 "순회"하기 위해서 이러한 레코드의 특성을 응용할 수 있습니다. 물론 타입에는 흔히 사용하는 `for`나 `while` 같은 반복문이 없기 때문에 재귀를 사용합니다.

이전 섹션에서 배운 내용들을 응용해서 재귀를 통해 타입을 "누적" 하기 위한 유틸리티 타입을 하나 정의합니다.

```typescript
type Append<Tuple extends readonly unknown[], Item> = (
  ((head: any, ...tail: Tuple) => void) extends ((...extended: infer Extended) => void)
  ? {
    [Index in keyof Extended]: Index extends keyof Tuple
      ? Tuple[Index]
      : Item
    }
  : never
);

type Prepend<Tuple extends readonly unknown[], Item> = (
  ((head: Item, ...args: Tuple) => any) extends ((...args: infer Result) => any)
    ? Result
    : never
);

// Does work well in v3.9.2, but doesn't in v4+
// $ExpectType [1, 2, 3, 4]
type Result0 = Append<[1, 2, 3], 4>;

// $ExpectType [0, 1, 2, 3, 4]
type Result1 = Prepend<Result0, 0>;
```

앗... 타입스크립트 버전이 4.0이 되면서 동작이 바뀌었는지 `Append`가 의도한대로 동작하질 않네요 ㅠㅠ 다음 섹션에서 새 기능을 이용해 고치는 방법을 설명드리겠습니다.

일단은 `Append` 대신 `Prepend`를 사용해보겠습니다. 이러면 누산한 결과가 뒤집히게 되기 때문에 이를 다시 뒤집어서 원래 의도한 결과대로 만들기 위해 추가적인 유틸리티를 정의하겠습니다.

```typescript
type Reverse<Tuple extends readonly unknown[], Result extends readonly unknown[] = []> = {
  finish: Result,
  step: Reverse<DropFirst<Tuple>, Prepend<Result, Tuple[0]>>,
}[Tuple['length'] extends 0 ? 'finish' : 'step'];

type ReversedDropLast<Tuple extends readonly unknown[], Result extends readonly unknown[] = []> =
{
  finish: Result,
  step: ReversedDropLast<DropFirst<Tuple>, Prepend<Result, Tuple[0]>>,
}[Tuple['length'] extends 1 ? 'finish' : 'step'];

// 이렇게 재귀를 포함한 타입정의를 중첩하면 타입스크립트의 휴리스틱이 올바르게 동작하지 않는 경우가 많습니다.
// 하지만 실제로는 종료조건만 잘 포함되어 있다면 타입이 성공적으로 추론되므로 해당 라인에 @ts-ignore를 표시해 에러를 무시해버릴 수 있습니다.
// (제가 아는 한 이게 유일하게 올바른 ts-ignore 활용법입니다.)
type DropLast<Tuple extends readonly unknown[], Result extends readonly unknown[] = []> =
  // @ts-ignore
  Reverse<{
    finish: Result,
    step: ReversedDropLast<DropFirst<Tuple>, Prepend<Result, Tuple[0]>>,
    //                                  ~~~~~~~~~~~~~~~~~~~~~~~~~~~
    //                                                             ^_____ Type does not satisfy the constraint 'readonly unknown[]'.(2344)
  }[Tuple['length'] extends 1 ? 'finish' : 'step']>;
// ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
//                                               ^____ Type instantiation is excessively deep and possibly infinite.(2589)

// 근데 이렇게 중간 변수를 활용하면 또 뭐라고 안합니다;
type _WeekDay_1 = ReversedDropLast<typeof days>;
type _WeekDay_2 = Reverse<_WeekDay_1>;
                            
// $ExpectType ['mon', 'tue', 'wed', 'thu', 'fri']
type WeekDay = DropFirst<DropLast<typeof days>>;
```

앞 뒤로 붙어있는 요소를 하나씩 잘라내어 주말이 아닌 요일만 포함하는 튜플 타입을 얻었습니다!

예제에서 활용한 `Reverse` 와 `DropLast`를 자세히보면, 프로그래밍 처음 배울때 잠깐 지나간 재귀식과 생긴게 똑같다는 사실을 발견할 수 있습니다.

일단 재귀함수의 구성요소들을 다 가지고 있습니다. 값을 누적하기 위한 default parameter(`Result`)를 하나 정의하고, `finish`와 `step`이라는 필드로 구성된 레코드를 사용했습니다.

매 `step`마다 주어진 `Tuple` 파라미터를 하나 잘라서 원하는 타입을 만들어 `Result` 파라미터에, 나머지는 다시 `Tuple` 파라미터로 사용하여 재귀하고 있으며, `Tuple['length']`에 대한 제약사항을 종료식으로 사용하고 있습니다. 종료조건에 다다르면 `finish` 필드를 통해 `Result`에 누적한 타입을 얻게 됩니다.

와우, 이 정도면 정말 타입 레벨에서 프로그래밍이 가능하군요!

## Variadic Tuple (+ v4.0)

가능한건 알겠는데 여러 제약사항을 피해다니느라 정의가 너무 크고 비직관적입니다.

저도 직접 필요한 타입을 몇 번씩 작성해보고 나서야 겨우 읽고 이해되기 시작했습니다. 불현듯 복잡한 정규표현식이나 Perl 코드를 일컫는 `Write-only code` 라는 표현이 떠오릅니다.

특히나 이런식으로 튜플과 관련된 타입을 다룰 때 쉽게 복잡해지기도 하고, 타입스크립트 휴리스틱에 의한 버그를 피하느라 `ts-ignore` 를 남발하게 되기도 합니다. `ts-ignore `의 문제점은 버전 변경 등으로 해당 코드가 동작하지 않게 되었을 때 감지할 수단이 없다는 점입니다. (거짓말입니다. [dtslint](https://github.com/microsoft/dtslint) 같은 도구로 "테스트 코드"를 작성하면 됩니다.) 

타입스크립트는 버전 4.0에서 이런 튜플과 관련된 문제를 지원하기 위해 Variadic tuple 이라고 불리는 새로운 문법을 추가했습니다.

Variadic tuple은 튜플 타입에 대한 비구조화(rest/spread)를 지원해서 앞서 함수 시그니쳐를 응용하던 방법보다 훨씬 쉽게 유틸리티 타입들을 구현할 수 있게 해줍니다.

```typescript
type Append<Tuple extends readonly unknown[], Item> = [...Tuple, Item];
type Prepend<Tuple extends readonly unknown[], Item> = [Item, ...Tuple];

type DropFirst<T extends readonly unknown[]> = T extends readonly [any?, ...infer U] ? U : [...T];
type DropLast<Tuple extends readonly unknown[]> = Tuple extends readonly [...infer U, any?] ? U : [...Tuple];
```

설명을 얹을 게 없을 정도로 간결하네요... `Append`와 `Prepend` 구현은 아주 직관적으로 바뀌었고, 표현이 넓어져서 `DropLast`를 구현하는데 재귀가 필요하지 않습니다.

이렇게 간결하고 직관적으로 바뀌니 앞서 했던 난해한 삽질들은 뭐였나 하며 허탈하기까지 합니다.

Variadic tuple에 대한 구체적인 설명과 유즈케이스들은 [이를 실제로 구현한 PR](https://github.com/microsoft/TypeScript/pull/39094)에서 가장 잘 설명하고 있습니다. 그 밖에도 [공식 예제](https://www.typescriptlang.org/play?ts=4.0.2#example/variadic-tuples)와 [릴리즈 노트](https://devblogs.microsoft.com/typescript/announcing-typescript-4-0-beta/#variadic-tuple-types)에 설명이 잘 나와있습니다.

## Flow의 $TupleMap 시뮬레이션 하기

Flow 쓰다가 TypeScript로 넘어오고나서 익숙해지는 동안 불편한 것들이 좀 많았는데, 그 중에서도 [`$TupleMap`](https://flow.org/en/docs/types/utilities/#toc-tuplemap)의 부재가 컸습니다.

<blockquote class="twitter-tweet"><p lang="ko" dir="ltr">드디어 Flow로 하는 것 중 TypeScript에서 안되는 거 찾았다. TupleMap 대안이 없어서 튜플 엔트리마다 제네릭 타입 씌우는 게 안되네</p>&mdash; Hyeseong Kim (@KrComet) <a href="https://twitter.com/KrComet/status/1030466278824136705?ref_src=twsrc%5Etfw">August 17, 2018</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

($TupleMap 기능은 [이전 포스트](/posts/flow-type-level-func/#objmap-tuplemap)에서도 간단히 소개한 적 있습니다.)

2.8 버전에 대망의 Conditional type & infer 기능이 추가되고 나서부터 바로 재귀를 이용한 추론이 가능했는지는 모르겠지만, 당시에는 이런 아이디어를 배우기 전이였으므로 그냥 타입스크립트가 조금 구린걸로 치고 넘어갔었죠.

하지만 지금은 타입스크립트를 더 잘 사용할 수 있잖아요? `[1, null, string, undefined]`를 `[Promise<1>, Promise<null>, Promise<string>, Promise<undefined>]` 로 매핑하는 것 쯤은 껌이잖아요?

```typescript
type TupleMapPromise<Tuple extends readonly unknown[], Result extends readonly unknown[] = []> = {
  finish: Result,
  step: TupleMapPromise<
    DropFirst<Tuple>,
    Append<Result, Promise<Tuple[0]>>,
  >,
}[Tuple['length'] extends 0 ? 'finish' : 'step'];

type t = [1, null, string, undefined];

// $ExpectType [Promise<1>, Promise<null>, Promise<string>, Promise<undefined>]
type r0 = TupleMapPromise<t>;
```

가능은 한데, Flow의 `$Call`과 같이 Type-leve application 구현이 가능한 수준은 아니라서 필요할 때마다 이런 재귀 타입을 작성해줘야 하는 것이 여전히 걸립니다.

몇 가지 제약사항을 임의로 건다면 더 편하게 재사용할 수 있는 유틸리티를 만들 수 있습니다. 가령 타입 파라미터를 하나만 받는 Nominal 타입들에 대해서만 제약한다면 알려진 타입을 모아 미리 구성된 유틸리티를 만들 수 있습니다.

```typescript
// 잘 알려진 Nominal 한 타입들
type BoxType<T> = (
  | Promise<T>
  | Set<T>
  | Array<T>
);

type Wrap<T, Box extends BoxType<any>> = (
  Box extends Promise<any> ? Promise<T> :
  Box extends Set<any> ? Set<T> :
  Array<T>
);

type Unwrap<T extends BoxType<any>> = (
  T extends Promise<infer U> ? U :
  T extends Set<infer U> ? U :
  T extends Array<infer U> ? U :
  never
);
```

물론 Flow만큼 범용적이거나 확장 가능하진 않지만, 그래도 비슷한 사례들은 한꺼번에 모아 유틸리티를 만들 수 있게 됐습니다.

```typescript
type TupleMapWrap<
  Tuple extends readonly unknown[],
  Box extends BoxType<any>,
  Result extends readonly unknown[] = []
> = {
  finish: Result,
  step: TupleMapWrap<
    DropFirst<Tuple>,
    Box,
    Append<Result, Wrap<Tuple[0], Box>>
  >,
}[Tuple['length'] extends 0 ? 'finish' : 'step'];

type TupleMapUnwrap<
  Tuple extends readonly BoxType<any>[],
  Result extends readonly unknown[] = []
> = {
  finish: Result,
  step: TupleMapUnwrap<
    DropFirst<Tuple>,
    Append<Result, Unwrap<Tuple[0]>>
  >,
}[Tuple['length'] extends 0 ? 'finish' : 'step'];

// $ExpectType [Promise<1>, Promise<null>, Promise<string>, Promise<undefined>]
type r1 = TupleMapWrap<t, Promise<any>>;

// $ExpectType [1, null, string, undefined]
type r2 = TupleMapUnwrap<r1>;
```

뭐 이 정도면 현업에서 쓰긴 충분한 것 같아요 (아마....?)



## 마무리하며

글에 소개된 예제들은 [Playground에서 가지고 놀아볼 수 있습니다](https://www.typescriptlang.org/play?ts=4.0.2#code/PTAEBIFEA8AcFMDGAXAKgTwaATvAhgCYD2AdgDbqgDaA5AM4CuJNANKDQLamvvIPw8aAd3gFByABYNBAM2wBLQXTzIaAXQBQiUnWSgCedHVABeavSaCuzNjT4DbIsbcnTbcxbeWq1oPMe0SXQBuDQ0DIyoABjVQiLoqAEZY8MMEgCYU+KoAZiy0qgAWfMiAVhKEgDYUkFAAAWQ6AFp4OCRkFuxsImxUyIB2FI1kTHhQADF5bF1IEmRsdAAeVFBW5HgSAmNcQlIKUCYAaxIiIRIqNQA+U1BUaJSRrAAZDYBzSWXV6HXN7fxiciUI4nM4Xa5mO40MhvSTqUIaWqAQBrQIAPccAjs2AEqHACctgA1x0CAABrAKXjoEAPuOADB7AIATKNAgFQ1wA7CwAKABG8F0oG0HGsoEe8AAlIABcdAJAYHBZ2FAgFDxwATTYATpoAdAiwFA2igMFgLMxQAAfdjWGja3j8fU64SiY28KTmmgeK3eGjDUagAAihhuPKIMn0BWFovg2CGtWVCFVjo19p5EymuhdlDMk2myFm8yW7s98Uu8MDMGDaEd-QdWAAwkQmMgAPIyGPGMwvEjvCSLVNeowZsIRp3dWDx3TLBiwaFfH5bHD-PZAkjHU7nK6mDSgUAMhly5fIPDyMgALlufehvNM1zwJHQe7WG2Hi4k-y3h-QbGXctX6638hIMj9tzXZD3JgPR95c9AAB+D911ALcSHgAA3P14QjJ5-GQXt+zGU9fhHXZAQOCcQWncFt2QqgOyILso0Q1Ad3gS5aGhOtYTUAMlWzdo1TGegVHDR14OjV0awQxtRg9Zs6FbRVQByOUAE45XSQAF0cAHEHABSm0BAE3mwBE8cADVXAAHJwAZztAQo5SiGTAAJB0BABwJwAAZsAHQ6VI0zTQEAKVHABlxgsxgAQVgBBNiQgdUOHHYAX2YEpwuNgAEl1g4PCGQAi8rz8I87xXT8t3I5Dv2uSCiHkAgT2+M9jEXe9fNEZ9X3fGAh1EdLQEy7L-3nYCAG8APnKhQs2VpQBfUBDngdBBIqs9RDULd2oITrfOMXr+s9VLoRa+cgPw6E2o66BNEWxbRoihaAF8APAqC-Q0XlYMdAAFXBPIIbyULytD-LHLDJ1BNQwoiqKYoZS9CG2+AOESuU8GwV46BSijqpvXKhwKpdl2B0HSrfcUACVWQYMhkEhv8FuAtHGExhbDug3pTrbR00ZJuh4Fuwd8vQgLxxe6c2HxjG9EmhmnqC16bjBG5mvnGQX3kOgJC3NnMZYADdHgWAJaO6YaaIkiE1uy42EuuWz0WSXkDYOb4HuS4NY0XaqEN6iYQkdQ6bQqIlutEWxf1Ld6HWWA4XJrBKb9amCBVriyIou2-NHTCeZZ0A9dDv4MMC7Dgt8Mx+ZMDRBdAYWSFF8Xo-RqWZY9hWqdEQO+JV7tg+QjXQC167dfz-XlqNmITel83Laha3bc5xJHaznPXfYWXPYY0BABdxwBwDsAGJrQEAGvHAAHa-FQEAG+XAAtVwAcFtAQABhYlQBUCZxQAfTtAQBEScASpn1MAFy6d4lDFABsFwBezsADqXQEAFnXABrOjFAEel8fQEAGjHAAoPYAF07Z5qS0g5JSgBemsAA1jy9ABznYAVqHZQKlqFpXSgATodMpZJSgBQicADMdgAHCcAC41v9AAZ46vNegAMFsAC2joBAAR4zKa+u9BSAEZBwArzUUkADzj1lAAps4AAw7KGAAY60AgAZVsABzdMlQCABdVwAn02gEAD7tKJAAi4-URoTR5CvBOLgUA1kZKABv24+gAWbsADtDUjAAhPYAFs7QCAAwh+hSD5QuWdJ2IOtNOaPQjonV6rNG6xy5h45mFw+YzjTvOWoDRmgaK0fAACvslaLAzkLZ2uc9bS02iPYuftS7OPLp2Su6tNZXR1ik5uxtTabVqItAAftUmptS6n1IaY0ppzTqkLQqZtDpnSundJ6b03pAA9AA+sM4ZtxHTEFZEKIgehvCixkJQSQYxAi6GwGuOY7B3EJwCWoGgcoGTpByIUQo9VQAdwolbWiNtfC937kkoe7s5bqFbLUFpry3nNNEn0r53yBkjMGWMrAL5dCHmQPIFQ8hSBdWMK0RArI6DyGgvscact4oEFALAIgdB4VMn2C+Ae6w9npFKAADgkv+USgAf2sAA7NE8Z6n0ACA1oBAADPdY4+gAdNcAJVjV9QCAAQ20AgAFbvkYAA5rQCAAlRwAlquylCBGQZAB1eA8BDgxkGX3MwsT-Zlx7E2dM0rHRyoVUqwwgz0g3HVTTfVirlWJFbD8n5okgzMUdLQPULgjSODNC4S07gFDqEcfKy1PEnHETyZqxC2q0gm0zGAFiABlRAChYB6H0lEPEkFgZgoIPIRA3IQ6CMAKVNKjAA1AxywAOqugEADa1gAQNY0CQPAHBWSwDwLC0AAA1dNhAs2G1ABnVoGLsB6AjO5euXa3Hhy2ThEKoBwr-TwlQe8ht3r-RSPOXtPQB0XUKV5Ed90w7xyZhOt6U6Pp82nQDUA86KJDBXXANd3JHQV1Ip8Ude7noHrwisZ9jNqA3kAoDPF74ACqvhgIAbAkKRWoRr19vXVgUNrid1xy-ZHVOzdfGbMoHO5c-7xQAbYD+4DoBQPExgmEUJjEVS5iwM67grqHDsCcOIL17AbReHYhtO9WBkAOzMA+tW4aWzwjIxAJiIYqOJDYOkNgOQ2DFAAhGZAqrQBDp1kkCTUnD2FBEkJh1omxjRDYOJ0AkmxIyfY-Jk1Zg646wU2wKIWnQBZgoyxcwLrDR0dNM4C0bhmM+rM46f1hrYxBtVj2OD-HhKtn2o4w2ABZPAsBLpEA4KLGm26YZ+PHUnbxBMOYIYy-upOQS8IZwHmLBWOXUmgHSc3OLCXujJeposBabaFAdsQIbOUvGeyGzKYtFrGbO0UTlMpryxTEsNdS+c1ulwAKmzOQRLulye55YdsBJ22cXZgYeaPKNwmnNOvGylxY1qClJaO8KMgZAa6Hcaysl8rxrv1aO0wcaWdRCXE0BGbA3GavxZuzTVTQoMZkDYHdusbAXvwDewQK4u3yGABlRwAJB2ABHJ0AAA5M7tayCgDLbvLEjiABCRBoAsWWJ9ecOp-tk4AjqGN8AyIzYp0prohhqdkz9as2Ayw2BE+gL43npObwmxuNFecvPfFU6F0tKnqBrgbgAuLzmdPEJS+AsrsnYEAKuRZ0sWXJ0zpYAAyQIQnOn15YF6MRYQvye3Al09xr2HCPXBA5r+cH68vq8dwB53hHXe285tr1ZSwvc+6IwBCCJN9fezGLF+LsrTcATS-TdDr6ssK+J-z4ngujy9Zjp+7mnjpxBI0MVgCpXkmN0q9V2PsB4-xaa5tfrbWOtdartCXrYvieVfnE3zN7WhsjZusUuvXPLYxB58Tk2s324W3OYt+sy30ureY3crbI8vbRYorVo3Jv6+J5DvnzCFuEBW5zyFGJPjD+Zd5inTQpfEkbYrxVwuctwbIW38bhPjf2195b7kx9PW3eraP+g2yEw2HkRSlehGn+9eY+VwjOoAc2s+C2NEC+1yK2tyj+9y6+Y8OmlGemVOJ2tc9uNMF2V2p2E2iwYOD2FBz2HU0OH2ji2AimNeI+iwgOZBoO8w92EO9BL4w0tBjW1uu2eBzmHBwOXBCg4OWEr2-BMOTBFmv2sAO+puzBGYQAA).

저는 이런 유틸리티들을 현업에서도 많이 사용하는 편이라 아예 [라이브러리](https://github.com/cometkim/cometjs/blob/master/packages/core/src/tuple.ts)를 만들어 모아두고는 합니다.

사실 타입수준 유틸리티들을 구현하고 있는 많은 라이브러리들이 이미 존재하고, 저도 이런 라이브러리들에서 코드를 훔치면서 이런 테크닉들을 배울 수 있었습니다.

- [utility-types](https://github.com/piotrwitek/utility-types)
- [ts-toolbelt](https://github.com/millsp/ts-toolbelt)
- [type-fest](https://github.com/sindresorhus/type-fest)
- [$mol_type](https://github.com/eigenmethod/mol/tree/master/type)

범용적인 유틸리티들은 이런 라이브러리에서 가져다가 쓰면 되지만, 그래도 직접 라이브러리 코드를 작성할 때면 종종 복잡한 타입을 작성하게 되는데 이런 테크닉들을 미리 배워두면 많은 도움이 되는 것 같습니다.

정확하게 작성되어 추론이 가능한 타입은 주변 개발자를 행복하게 합니다. 하지만 2시간 동안 타입 시그니쳐와 씨름하느라 실제로 돌아가는 코드를 작성하지 못했다고 하면 관리자는 화를 낼 수도 있습니다.

TypeScript의 타입시스템은 정말 강력한 마법이면서 동시에 헤어나기 힘든 늪과도 같습니다.

애초에 nominal 한 객체와 일관성 있는 인터페이스를 활용한다면 이런짓을 할 일이 없긴 하겠지만, 뭐 이건 JavaScript의 저주라고 해두겠습니다. :wink:

그런 의미에서 가장 유용한 유틸리티 타입을 소개드리며 마무리합니다.

```typescript
type $FixMe = any;
type $FixMeInTomorrow = any;
type $FixMeInNextSprint = any;
```

