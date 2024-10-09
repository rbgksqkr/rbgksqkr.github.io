---
title: "[React] hydration"
layout: single
excerpt: React에서 Hydration이란 무엇이며, 동작 방식에 대해 설명할 수 있다.

categories:
  - react
tags: [SSR, hydration, render]
toc_label: hydration
toc: true
toc_sticky: true
---

## 💭 예상 질문

- React에서 Hydration이란 무엇인가요?
- SSR과 CSR에서 Hydration의 차이점은 무엇인가요?
- Hydration이 동작하는 과정을 설명해 주세요.
- Hydration 과정에서 발생할 수 있는 문제는 무엇이며, 이를 어떻게 해결할 수 있나요?
- React에서 <span class="highlight">hydrate()</span>와 <span class="highlight">render()</span>의 차이점은 무엇인가요?

<div class="red-box">
    <div>hydration은 서버에서 만든 정적인 HTML에 React 컴포넌트 로직을 결합하여 상호작용이 가능하도록 만드는 과정입니다.</div>
    <div>서버에서 만든 HTML은 이벤트가 하나도 없는 정적 파일인데, <span class="highlight">hydration을 통해 상호작용이 가능</span>해집니다.</div>
    <br />
    <div>주의할 점은 <span class="highlight">서버에서 렌더링한 HTML과 클라이언트에서 렌더링한 가상 DOM이 일치</span>해야 hydration이 정상적으로 이뤄집니다.</div>
    <div>두 트리가 일치하지 않을 경우 화면을 다시 그리는 성능 문제나 애플리케이션이 동작하지 않는 에러가 발생할 수 있습니다.</div>
</div>

## 🔍 추가 내용

### ✅ hydration 동작 과정

<div class="blue-box">
  <div>1. 브라우저에서 SSR 서버에 HTML 요청</div>
  <div>2. SSR 서버에서 API 서버에 데이터 요청</div>
  <div>3. SSR 서버는 API 서버에서 받은 데이터를 HTML에 담아 브라우저에 응답</div>
  <div>4. 브라우저는 받은 HTML에 React 컴포넌트 로직을 붙이기 위해 hydration</div>
  <div>5. hydration을 통해 서버에서 만들어진 최초의 HTML에 이벤트를 붙여 동적인 웹 사이트로 동작</div>
</div>

### ✅ hydration 에러 발생 원인

주로 아래와 같은 원인으로 hydration 에러가 발생한다고 한다.

<div class="blue-box">
  <li>React를 통해 만들어진 HTML의 root node 안에 줄바꿈같은 추가적인 공백</li>
  <li>`typeof window !== 'undefined'` 과 같은 조건을 렌더링 로직에서 사용</li>
  <li>`window.matchMedia` 같은 브라우저에서만 사용가능한 API를 렌더링 로직에 사용</li>
  <li>서버와 클라이언트에서 서로 다른 데이터 렌더링</li>
</div>
### ✅ 서버에서 렌더링한 HTML과 클라이언트에서 렌더링한 가상 DOM이 왜 일치해야 하는가?

hydrate할 때 <mark class="mark">서버에서 렌더링된 정적 HTML과 클라이언트에서 렌더링되는 React의 가상 DOM을 비교하여, 변경된 부분만 최소한의 업데이트로 상호작용을 추가</mark>한다.

HTML과 가상 DOM이 일치하지 않으면, React는 <mark class="mark">HTML을 다시 그리거나</mark> 기존 콘텐츠를 덮어쓴다. 이것은 성능적으로 비효율적일 뿐 아니라, 레이아웃이 변경되어 UX에 악영향을 줄 수 있다.

## 📘 reference

[React 공식문서 - hydrateRoot](https://ko.react.dev/reference/react-dom/client/hydrateRoot)
