---
title: Babel 7로 Node.js에서 모던 자바스크립트 시작하기
author: Hyeseong Kim
tags:
  - babel
  - javascript
  - node
references:
  - https://www.slideshare.net/ssuser2295821/babel-7
  - http://ahnheejong.name/articles/ecmascript-tc39/
---

> 주의사항: 이 글은 현재 베타버전인 Babel 7에서의 사용법을 기준으로 설명한다.

# 바벨이란?
바벨은 자바스크립트 표준인 ECMAScript(이하 ES)의 최신 문법으로 작성된 코드를 실행할 수 있는 이전 버전 문법으로 변환해주는 트랜스파일러이다. (core-js 등의 주요 폴리필도 포함하고 있다.)

자바스크립트는 [ECMAScript 표준과, TC39 위원회](http://ahnheejong.name/articles/ecmascript-tc39/), 자바스크립트 커뮤니티를 통해 빠르게 발전해오고 있지만 브라우저들과 Node.js가 지원하는 자바스크립트는 이를 완전히 따라잡지 못하고 있다. 특히 구버전의 IE를 지원하거나 해야되는 상황도 많기 때문에 순수히 ES를 사용해 개발하는 것은 그림의 떡같은 상황이다.

> [ES 호환성 테이블](http://kangax.github.io/compat-table)에서 지원현황을 확인할 수 있다.

최신(Current)버전의 노드는 비교적 문법지원이 빠른 편이지만, 서버리스 플랫폼을 사용하는 등 이유로 트랜스폼이나 폴리필이 필요할 수도 있다.

바벨을 사용하면 이런 호환성 걱정 없이 자바스크립트의 최신 문법을 어디서나 자유롭게 사용할 수 있고, 이 후에 포함될 문법도 미리 사용해볼 수 있다.

# 바벨 설정하기
이미 ES 문법을 사용하거나 리액트를 사용하는 프로젝트라면 이미 설정되어 있겠지만, 간단하게 설치하는 법과 설정하는 법을 살펴본 후 어떻게 **최적의 설정** 하는지 살펴보자.

먼저 Yarn을 이용해 AwesomeProject라는 이름으로 프로젝트를 생성하고, 바벨을 사용하기 위한 필수 패키지를 설치한다.

```sh
# Yarn으로 Node 프로젝트 생성
mkdir AwesomeProject
cd AwesomeProject
yarn init -y

# 바벨 패키지 설치
yarn add --dev @babel/core @babel/cli @babel/preset-env @babel/polyfill
```

프로젝트 루트에 간단하게 `package.json`이 생성되고 `@babel/core`, `@babel/cli`, `@babel/preset-env` 패키지가 devDependencies에 추가된다. 폴리필 사용도 설명하기 위해 `@babel/polyfill`도 추가했다.

동작을 확인하기 위해 일단 간단한 코드를 `index.js`라는 이름으로 생성한다.

```js
import '@babel/polyfill'

const name = 'World'

const main = async () => {
    console.log(`Hello ${name}`)
}

main().catch(console.error)
```

[ES모듈 임포트](https://nodejs.org/api/esm.html), [화살표 함수](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions), [템플릿 리터럴](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Template_literals), [Async 함수](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Statements/async_function) 등의 문법을 사용했다.

이 파일을 설치한 바벨 CLI를 이용해서 트랜스파일 해보자

```sh
npx babel --presets @babel/env index.js
```

트랜스파일 된 코드가 stdout으로 다음과 같이 출력된다.

```js
"use strict";

require("@babel/polyfill");

function _asyncToGenerator(fn) { return function () { var self = this, args = arguments; return new Promise(function (resolve, reject) { var gen = fn.apply(self, args); function step(key, arg) { try { var info = gen[key](arg); var value = info.value; } catch (error) { reject(error); return; } if (info.done) { resolve(value); } else { Promise.resolve(value).then(_next, _throw); } } function _next(value) { step("next", value); } function _throw(err) { step("throw", err); } _next(); }); }; }

var name = 'World';

var main =
/*#__PURE__*/
function () {
  var _ref = _asyncToGenerator(
  /*#__PURE__*/
  regeneratorRuntime.mark(function _callee() {
    return regeneratorRuntime.wrap(function _callee$(_context) {
      while (1) {
        switch (_context.prev = _context.next) {
          case 0:
            console.log("Hello ".concat(name));

          case 1:
          case "end":
            return _context.stop();
        }
      }
    }, _callee, this);
  }));

  return function main() {
    return _ref.apply(this, arguments);
  };
}();

main().catch(console.error);
```

`import` 구문이 commonjs의 `require`로 바뀌고 `async`에 대한 폴리필이 들어가는 등, 언급한 문법들이 조금 더 하위 문법(ES5)의 코드로 변환된 것을 확인할 수 있다.

이렇게 생성된 코드는 노드 버전 4에서 실행하더라도 의도한대로 잘 동작한다. (정확히는 유지보수되고 있는 노드 버전에 대한 동작을 보장한다. 버전 5는 LTS버전이 아니라서 이미 EoL이 지났기 때문에 공식적으로 지원하진 않는다.)

# Env 프리셋 사용하기

잠깐, 위 예제에 치명적인 문제가 있다.

# 모던 자바스크립트 사용하기