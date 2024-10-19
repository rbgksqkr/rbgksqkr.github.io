---
title: "[JS] 모듈 시스템"
layout: single
excerpt: 모듈 시스템을 이해하며, ESM과 CommonJS의 차이를 설명할 수 있다.

categories:
  - javascript
tags: [ESM, CommonJS, 모듈, 비동기 로드, tree-shaking]
toc_label: 모듈 시스템
toc: true
toc_sticky: true
---

## 💭 예상 질문

- 자바스크립트 모듈 시스템에 대해 설명해주세요.
  - 모듈 시스템을 왜 사용하나요?
- ESM과 CommonJS의 차이점은 무엇인가요?
- 브라우저에서 ESM 모듈을 어떻게 사용하나요?
- 모듈 번들러의 역할은 무엇인가요? (webpack 등)

<div class="red-box">
  <div>자바스크립트 모듈 시스템은<mark class="mark">코드를 작게 분리하고, 필요한 모듈만 가져와 사용하여 재사용성과 유지보수성을 높이는 시스템</mark>입니다.</div>
  <div>모듈 시스템은 크게 CommonJS와 ESM으로 구분합니다. CommonJS는 require()과 module.exports 로 모듈을 가져오고 내보내며, ESM은 import와 export 키워드를 사용합니다.</div>
  <br />
  <div>브라우저 환경에서는 ESM이 표준이며 <mark class="mark">모듈을 비동기적으로 로드</mark>하고, CommonJS는 Node.js 환경에서 주로 사용되며<mark class="mark">모듈을 동기적으로 로드</mark>합니다.</div>
</div>

### ✅ 동기 로드 & 비동기 로드

**🔥 CommonJS**

- 모듈을 동기적으로 로드하여, 모듈이 로드될 때까지 코드 실행이 중단된다.
- 모든 모듈이 로드된 후에 코드가 실행되는 서버 환경에서 유리하다.

```js
// add.js
module.exports.add = (x, y) => x + y;

// main.js
const { add } = require("./add");

add(1, 2);
```

**🔥 ESM (EcmaScript Modules)**

- ESM은 [Top Level Await](https://nodejs.org/api/esm.html#top-level-await) 을 지원하기 때문에 모듈을 비동기적으로 로드한다.
- 브라우저에서 모듈을 로드할 때 <mark class="mark">동기적으로 로드된다면, 브라우저 렌더링을 막아 성능 저하를 일으킬 수 있다.</mark>

```js
// add.js
export function add(x, y) {
  return x + y;
}

// main.js
import { add } from "./add.js";

add(1, 2);
```

### ✅ tree-shaking

**🔥 CommonJS**

- <mark class="mark">CJS는 트리 쉐이킹이 어렵고, ESM은 쉽게 가능</mark>
- CJS는 기본적으로 `require / module.exports`를 동적으로 사용하는 것에 아무런 제약이 없다.
  - 따라서 CJS는 빌드 타임에 정적 분석을 적용하기 어렵고, 런타임에서만 모듈 관계를 파악할 수 있다.

```js
// require
const utilName = /* 동적인 값 */
const util = require(`./utils/${utilName}`);

// module.exports
function foo() {
  if (/* 동적인 조건 */) {
    module.exports = /* ... */;
  }
}
foo();
```

**🔥 ESM (EcmaScript Modules)**

- ESM은 `정적인 구조로 모듈끼리 의존하도록 강제`한다.
- import 경로에 동적인 값을 사용할 수 없고, export는 항상 최상위 스코프에서만 사용할 수 있다.

```js
import util from `./utils/${utilName}.js`; // 불가능

import { add } from "./utils/math.js"; // 가능

function foo() {
  export const value = "foo"; // 불가능
}

export const value = "foo"; // 가능
```

<div class="blue-box">
  <div>따라서 ESM은 <mark class="mark">빌드 단계에서 정적 분석을 통해 모듈 간의 의존 관계를 파악</mark>할 수 있고, Tree-shaking을 쉽게 할 수 있다.</div>
</div>

## 📘 reference

- [토스 - CommonJS와 ESM에 모두 대응하는 라이브러리 개발하기: exports field](https://toss.tech/article/commonjs-esm-exports-field)
- [카카오 - CommonJS에서 ESM으로 전환하기](https://tech.kakao.com/posts/605)
- [https://ko.javascript.info/modules-intro](https://ko.javascript.info/modules-intro)
- [https://f-lab.kr/insight/commonjs-vs-esmodule-20240523](https://f-lab.kr/insight/commonjs-vs-esmodule-20240523)
