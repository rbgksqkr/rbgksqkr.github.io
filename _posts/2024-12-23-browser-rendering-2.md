---
title: "[Web] 브라우저 렌더링 (2) - HTML 파싱 및 렌더 트리"
layout: single
excerpt: 브라우저에 대해 이해하며, 어떻게 화면에 출력하는지 이해할 수 있다.

categories:
  - web
tags:
  [
    web,
    브라우저 렌더링,
    HTML 파싱,
    CSS 파싱,
    DOM,
    CSSOM,
    렌더 트리,
    fetch,
    async/await,
  ]
toc_label: 브라우저 렌더링 (2) - 파싱 및 트리 생성
toc: true
toc_sticky: true
---

<div class="blue-box">
  <div>2. 렌더링 엔진은 HTML과 CSS를 어떻게 해석하는지 살펴보자.</div>
</div>

브라우저의 구성 요소에 대한 내용은 [이전 글](https://rbgksqkr.github.io/web/browser-rendering-1/)에서 확인할 수 있다.

## 💭 글의 방향

웹서비스를 개발하면 브라우저는 <mark class="mark">HTML 파일을 가장 먼저 요청하여 화면에 출력</mark>한다.

<img width="500" alt="image" src="https://github.com/user-attachments/assets/6c2fecb4-5e58-426e-958f-c0437503db7b" />

`브라우저는 HTML 파일을 어떻게 이해하고 해석하는가?` 에 초점을 두었고, 브라우저 렌더링의 흐름을 어느 정도 알고 있다는 가정 하에 글을 작성하였다.

브라우저의 요청에 의해 웹서버가 HTML 문서를 응답하는데, 서버가 응답하는 HTML 문서는 문자열이다.

문자열을 브라우저에 시각적인 픽셀로 렌더링하려면 HTML 문서를 브라우저가 이해할 수 있는 자료구조로 변환하여 메모리에 저장해야 한다.

파싱 일부분을 유사하게 코드로 작성해봤는데, `렌더링 엔진에서 이뤄지는 작업을 코드로 표현` 했다고 이해하면 된다.

> How?

## 📘 HTML 파싱

전체적인 HTML 파싱 과정을 살펴보자.

1. 서버는 브라우저가 요청한 HTML 파일을 읽어 들여 메모리에 저장한 다음, 메모리에 저장된 바이트(2진수)를 인터넷을 경유하여 응답한다.
2. 브라우저는 <mark class="mark">서버가 응답한 HTML 문서를 바이트 형태로 응답받는다.</mark>
   1. 바이트 형태의 HTML 문서는 meta 태그의 charset 어트리뷰트에 의해 지정된 <mark class="mark">인코딩 방식(UTF-8)을 기준으로 문자열로 변환</mark>된다.
   2. 응답 헤더(response header)에 담겨 응답되며, 브라우저는 이를 확인하고 문자열로 변환한다.
   - ex) content-type: text/html; charset=utf-8
3. 문자열로 변환된 HTML 문서를 문법적 의미를 갖는 코드의 최소 단위인 <mark class="mark">토큰으로 분해</mark>한다.
4. 각 토큰들을 <mark class="mark">객체로 변환하여 노드를 생성</mark>한다. (문서 노드, 요소 노드, 어트리뷰트 노드, 텍스트 노드 등)
5. HTML 문서는 HTML 요소들의 집합으로 이뤄지며, HTML 요소는 중첩 관계를 갖는다.
   1. HTML 요소 간에 중첩 관계에 의해 부자 관계가 형성된다.
   2. HTML 요소 간의 <mark class="mark">부자 관계를 반영하여 모든 노드들을 트리 자료구조(DOM)로 구성</mark>한다.

### ✅ HTML 파일을 바이트 코드로 응답받아 문자열로 변환

브라우저에서 경로에 해당하는 index.html을 요청하듯이 fetch API를 활용하여 HTML 파일을 요청하였다.

`응답값을 바이트 코드로 읽고 디코딩하여 문자열로 읽는 과정` 을 코드로 옮겨보았다.

1. index.html을 fetch 함수로 요청하여 응답받는다.
2. 응답값을 바이트 코드로 읽어서 문자열로 변환한다.
   - ArrayBuffer + TextDecoder로 사용할 수도 있고, response.text()로 변환할 수도 있다.
   - ArrayBuffer로 다루면 인코딩 방식이 커스텀 가능하고, 바이너리 데이터와 함께 사용할 수 있다.

<img width="450" alt="image" src="https://github.com/user-attachments/assets/f939d4ce-bdf2-4bfa-90f0-b64b80a9b4ea" />

### ✅ 문자열로 변환된 HTML 문서를 토큰으로 분해하고 DOM 트리 생성

1. HTML 문자열을 문법적 의미를 갖는 코드의 최소 단위인 토큰으로 분해한다.
   - Doctype, 주석, HTML 태그, 텍스트로 총 4가지 타입의 토큰으로 분해하였다.
   - 실제론 더 다양한 타입의 토큰으로 분해되겠지만 렌더링 엔진을 만드는 게 아니니 4가지로만 분해하였다.
2. 각 토큰을 기반으로 노드 객체 생성
   - 기존의 토큰으로 분해했던 데이터를 트리 구조를 형성하기 위해 적합한 객체 형태로 변환한다.
3. 생성한 노드 객체로 DOM 트리 생성

<img width="600" alt="image" src="https://github.com/user-attachments/assets/8596dd76-7a1e-4399-8186-2e7b9db9bebc" />

## 📘 CSS 파싱

1. **CSS 코드 로드:** 브라우저가 HTML 파싱 중 `<link>` 또는 `<style>` 태그를 만나면 CSS 파일을 로드하거나, `<style>` 태그의 인라인 CSS를 바로 읽는다.
2. **바이트코드로 로드:** 외부 CSS 파일은 HTML과 마찬가지로 서버에서 바이트코드로 응답받는다.
3. **토큰화:** CSS 코드를 토큰으로 분해한다.
   - 예: `body { color: red; }` → `Selector: "body"`, `Property: "color"`, `Value: "red"`
4. **파싱 트리 생성:** 토큰화된 데이터를 바탕으로 CSS 규칙을 객체 형태로 변환한다.
   - 예: `{ selector: "body", declarations: { color: "red" } }`
5. **CSSOM 생성:** CSS 파싱 트리를 이용해 브라우저는 `CSSOM` 을 생성한다.

|                                                       출력 결과                                                       |                                                       CSS 파싱                                                        |
| :-------------------------------------------------------------------------------------------------------------------: | :-------------------------------------------------------------------------------------------------------------------: |
| <img width="400" alt="image" src="https://github.com/user-attachments/assets/5bfa67b0-a9fd-4f67-b9f6-ef0ff035482b" /> | <img width="400" alt="image" src="https://github.com/user-attachments/assets/606e7dba-9f75-41e0-9b34-c574ce54a874" /> |

## 📘 렌더 트리 생성

DOM과 CSSOM을 결합하여 렌더 트리를 생성한다.

모든 HTML이 파싱될 때까지 기다리지 않으며, DOM 트리가 생성되는 동안 브라우저는 렌더 트리를 생성한다.

렌더 트리는 시각적 요소가 표시될 순서대로 표시되며, 렌더 트리의 목적은 <mark class="mark">콘텐츠를 올바른 순서로 페인팅할 수 있도록 하는 것</mark>

렌더 트리의 요소: Firefox에서는 `프레임`, WebKit에서는 렌더러 또는 `렌더링 객체` 라고 표현한다.

각 렌더링 객체는 노드의 CSS box에 해당하는 직사각형 영역을 나타내며, 여기에는 너비, 높이, 위치와 같은 기하학적 정보가 포함된다.

box는 `display` 속성에 따라 DOM 노드에 생성할 렌더링 객체의 유형을 결정한다.

```tsx
class RenderObject{
  virtual void layout();
  virtual void paint(PaintInfo);
  virtual void rect repaintRect();
  Node* node;  //the DOM node
  RenderStyle* style;  // the computed style
  RenderLayer* containgLayer; //the containing z-index layer
}
```

## 💭 추가 학습 내용

HTML을 fetch하고 디코딩하는 과정에서 궁금했던 점을 비교하였다.

### response.text vs response.arrayBuffer + TextDecoder

**text**

- 브라우저가 응답의 `Content-Type` 헤더를 확인하여 인코딩 방식을 자동으로 판단하고, 해당 인코딩 방식에 따라 데이터 디코딩

**ArrayBuffer + TextDecoder**

- 바이트 코드를 다루므로, 로우 레벨에서 디코딩 방식을 명시적으로 정할 수 있음
- 다른 바이너리 데이터(이미지, 파일 등)와 같이 쓰기에 좋음

### **fetch 함수의 응답값을 반환할 때 await 을 붙인 것과 안붙인 것의 차이점은?**

- 공통점: 호출하는 곳에서 then() 또는 await을 사용하면 반환값은 `Promise<Response>` 로 같다.
- 차이점: <mark class="mark">에러가 발생했을 때 에러가 발생하는 위치가 다르다.</mark>
  - await 사용 O: fetch 함수 실행 결과를 기다리므로 `fetch 함수를 호출하는 곳에서 에러가 발생`한다.
  - await 사용 X: fetch 함수가 반환한 Promise를 그대로 반환하므로, `Promise를 처리하는 호출부에서 에러가 발생`한다.

### **fetch 함수를 호출할 때 async 키워드가 있는 것과 없는 것의 차이점은?**

- async를 안붙여도 fetch함수가 Promise를 반환하기 때문에 반환 타입은 `Promise<Response>` 다.
- async 키워드는 <mark class="mark">함수의 반환값에 영향을 미친다.</mark>

  - 따라서, async를 붙이지 않으면 에러가 발생했을 때 catch문에서 반환하는 값은 Promise가 아니다.
  - ex) async O: `Promise<Response | null>`
  - ex) async X: `Promise<Response> | null`

  ```js
  // 1. await을 붙인 것
  const getHTML = async () => {
    try {
      return await fetch("//");
    } catch (error) {
      console.log("함수에서 에러 처리");
      return null;
    }
  };

  // 2. await을 안 붙인 것
  const getHTML = async () => {
    try {
      return fetch("//");
    } catch (error) {
      console.log("함수에서 에러 처리");
      return null;
    }
  };

  // 3. async/await을 모두 안 붙인 것
  const getHTML = () => {
    try {
      return fetch("//");
    } catch (error) {
      console.log("함수에서 에러 처리");
      return null;
    }
  };

  getHTML()
    .then((result) => console.log(result))
    .catch(() => console.log("호출부에서 에러 처리"));
  ```

## 💻 브라우저 렌더링 예시 코드

코드가 길어서 따로 분리하였다.

DOM 트리 생성, CSSOM 트리 생성, 렌더 트리 생성하는 스크립트를 분리하여 작성하였다.

### html-parser.js

```js
// html-parser.js
const selfClosingTags = new Set(["meta", "img", "input", "link", "br", "hr"]);

const getHTML = async () => {
  return await fetch("/");
};

// 1. 바이트 코드를 HTML 문자열로 변환
const decodeHTML = async () => {
  const response = await getHTML();
  const contentType = response.headers.get("content-type");

  if (contentType && contentType.includes("charset=UTF-8")) {
    const buffer = await response.arrayBuffer(); // 바이트 데이터로 읽기
    const decoder = new TextDecoder("utf-8");
    return decoder.decode(buffer); // 문자열로 변환

    // return await response.text(); // 위의 3줄을 한줄로 사용 가능
  } else {
    console.error("Unsupported Content-Type or charset missing");
    return null;
  }
};

// 2. HTML 문자열을 토큰으로 분해
const tokenizeHTML = (htmlString) => {
  const tagRegex =
    /<!--[\s\S]*?-->|<!DOCTYPE[^>]*>|<\/?([a-zA-Z0-9\-]+)([^>]*)>|([^<]+)/g;
  const tokens = [];
  let match = tagRegex.exec(htmlString);

  while (match !== null) {
    if (match[0].startsWith("<!--")) {
      // Comment 토큰
      tokens.push({
        type: "comment",
        content: match[0].slice(4, -3).trim(), // 주석 내용만 추출
      });
    } else if (match[0].startsWith("<!DOCTYPE")) {
      // Doctype 토큰
      tokens.push({
        type: "doctype",
        content: match[0],
      });
    } else if (match[1]) {
      // 일반 태그 토큰
      tokens.push({
        type: match[0].startsWith("</") ? "closing-tag" : "opening-tag",
        tagName: match[1],
        attributes: match[2].trim(),
      });
    } else if (match[3].trim()) {
      // 텍스트 토큰
      tokens.push({
        type: "text",
        content: match[3].trim(),
      });
    }

    match = tagRegex.exec(htmlString);
  }

  return tokens;
};

// 노드를 생성할 때 속성 문자열을 객체로 변환
const parseAttributes = (attributesString) => {
  if (!attributesString) return {};
  return attributesString
    .split(/\s+/)
    .map((attr) => attr.split("="))
    .reduce((acc, [key, value]) => {
      acc[key] = value ? value.replace(/['"]/g, "") : ""; // 따옴표 제거
      return acc;
    }, {});
};

// 3. 각 토큰을 기반으로 노드 생성
const createNode = (token) => {
  if (token.type === "opening-tag") {
    return {
      type: "element",
      tagName: token.tagName,
      attributes: parseAttributes(token.attributes),
      children: [],
    };
  } else if (token.type === "text") {
    return {
      type: "text",
      content: token.content,
    };
  } else if (token.type === "comment") {
    return {
      type: "comment",
      content: token.content,
    };
  }

  return null;
};

// 4. DOM 트리 생성
const buildDOMTree = (tokens) => {
  const root = { type: "root", children: [] };
  const stack = [root];

  tokens.forEach((token) => {
    if (token.type === "opening-tag") {
      const node = createNode(token);
      if (node) {
        stack[stack.length - 1].children.push(node); // 부모에 추가

        if (!selfClosingTags.has(token.tagName)) {
          stack.push(node); // Self-closing 태그가 아니면 스택에 추가
        }
      }
    } else if (token.type === "closing-tag") {
      stack.pop();
    } else if (token.type === "text" || token.type === "comment") {
      const node = createNode(token);
      if (node) {
        stack[stack.length - 1].children.push(node); // 부모에 텍스트 추가
      }
    }
  });

  return root.children[0]; // 최종 DOM 트리 반환
};

// 5. DOM 트리 출력
const printDOMTree = (nodes, depth = 0) => {
  nodes.forEach((node) => {
    if (node.type === "element") {
      console.log(`${"  ".repeat(depth)}<${node.tagName}>`);
      printDOMTree(node.children, depth + 1); // 자식 노드 출력
      console.log(`${"  ".repeat(depth)}</${node.tagName}>`);
    } else if (node.type === "text") {
      console.log(`${"  ".repeat(depth)}${node.content}`);
    } else if (node.type === "comment") {
      console.log(`${"  ".repeat(depth)}<-- ${node.content} -->`);
    }
  });
};

const getDOMTree = async () => {
  const htmlString = await decodeHTML();

  if (htmlString === null) return;

  const tokens = tokenizeHTML(htmlString);
  return buildDOMTree(tokens);
};

export default getDOMTree;
```

### css-parser.js

```js
const cssCode = `
body {
  color: red;
  background: white;
}

h1 {
  font-size: 24px;
  margin: 10px;
}
`;

function tokenizeCSS(cssString) {
  const ruleRegex = /([^{]+)\{([^}]+)\}/g;
  const tokens = [];
  let match = ruleRegex.exec(cssString);

  while (match !== null) {
    tokens.push({
      selector: match[1].trim(),
      declarations: match[2].trim(),
    });

    match = ruleRegex.exec(cssString);
  }

  return tokens;
}

function parseDeclarations(declarationsString) {
  return declarationsString
    .split(";")
    .filter(Boolean)
    .reduce((acc, decl) => {
      const [property, value] = decl.split(":").map((str) => str.trim());
      acc[property] = value;
      return acc;
    }, {});
}

function buildCSSOM(cssString) {
  const tokens = tokenizeCSS(cssString);
  return tokens.map((token) => ({
    selector: token.selector,
    declarations: parseDeclarations(token.declarations),
  }));
}

const cssOM = buildCSSOM(cssCode);
export default cssOM;
```

### render.js

```js
import getDOMTree from "./html-parser.js";
import cssOM from "./css-parser.js";

// Render Tree 생성
async function buildRenderTree(cssOM) {
  const domTree = await getDOMTree();

  function applyStyles(node, inheritedStyles = {}) {
    if (node.type === "element") {
      const cssRule = cssOM.find((rule) => rule.selector === node.tagName);
      const styles = cssRule
        ? { ...inheritedStyles, ...cssRule.declarations }
        : inheritedStyles;

      if (styles.display === "none") {
        return null;
      }

      return {
        tagName: node.tagName,
        styles,
        children: node.children
          .map((child) => applyStyles(child, styles)) // 자식 노드에 동일한 스타일 적용
          .filter(Boolean),
      };
    } else if (node.type === "text") {
      return { content: node.content };
    }
    return null;
  }

  const result = applyStyles(domTree);
  console.log("직접 구현한 렌더 트리:", result);
}

const renderTree = buildRenderTree(cssOM);
```

### index.html

```html
<!DOCTYPE html>
<html lang="ko">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>브라우저 렌더링 과정</title>
    <link rel="stylesheet" href="/browser/style.css" />
  </head>
  <body>
    <!-- Comment here -->
    <div class="container" id="contaier__1">
      <h1>브라우저 렌더링 과정 파헤치기</h1>
      <p>This is a demo</p>
    </div>
    <div style="display: none">display:none</div>
    <script type="module" src="/browser/html-parser.js"></script>
    <script type="module" src="/browser/css-parser.js"></script>
    <script type="module" src="/browser/render.js"></script>
  </body>
</html>
```

## 📘 reference

- [how browsers work? - Chrome dev](https://web.dev/articles/howbrowserswork?hl=ko)
