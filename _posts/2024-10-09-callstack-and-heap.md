---
title: "[JS] 콜스택과 힙"
layout: single
excerpt: 콜스택과 힙은 무엇이며, 각각의 역할을 설명할 수 있다.

categories:
  - javascript
tags: [콜스택, 힙, 실행 컨텍스트, 싱글 스레드, 비동기, 이벤트 루프]
toc_label: 콜스택과 힙
toc: true
toc_sticky: true
---

## 💭 예상 질문

- 자바스크립트의 콜스택이 무엇인지 설명해 주세요.
  - 콜스택에 쌓이는 과정과 제거되는 과정을 간단히 설명해 주세요.
- 콜스택과 힙에 저장되는 데이터의 차이점은 무엇인가요?
- 자바스크립트 힙에서 가비지 컬렉션은 어떻게 작동하나요?
- 클로저에서 힙과 콜스택의 역할은 무엇인가요?
- 자바스크립트의 싱글 스레드 특성과 콜스택의 관계에 대해 설명해 주세요.
  - 콜스택과 힙이 과하게 사용되면 각각 어떤 문제가 발생할 수 있나요?

<div class="red-box">
    <div>콜스택은 <span class="highlight">함수들의 호출 순서를 제어하는 자료구조</span>입니다.</div>
    <div>함수가 호출되면 콜스택의 최상단에 쌓이고, 함수가 종료되면 콜스택에서 제거됩니다.</div>
    <br />
    <div>힙은 객체, 배열, 함수처럼 <span class="highlight">크기가 동적으로 변할 수 있는 참조타입 값을 저장하는 공간</span>입니다.</div>
    <div>콜스택과 달리 훨씬 큰 메모리 공간을 가집니다.</div>
    <br />
    <div>콜스택은 <mark class="mark">원시타입 데이터와 함수 호출의 실행 컨텍스트를 저장</mark>하며, 힙은 <mark class="mark">참조 데이터를 저장</mark>합니다.</div>
    <div>함수가 호출되면 콜스택에 실행 컨텍스트가 쌓이고, <mark class="mark">함수에서 객체나 배열을 만들면 힙에 할당</mark>됩니다. <mark class="mark">콜스택에선 힙의 메모리 주소를 참조</mark>하여 데이터를 처리합니다.</div>
</div>

## 🔍 추가 내용

### ✅ 싱글 스레드 성능 문제

<div class="blue-box">
  <li>재귀 호출 등으로 인해 함수 호출 수가 괴도하게 많아진다면 스택 오버 플로우가 발생한다.</li>
  <li>한 번에 하나의 함수만 처리할 수 있다면 한 함수의 작업량이 많다면 다른 함수 호출에 영향을 미친다.</li>
  <li>자바스크립트에서는 비동기 콜백을 통해 오래걸리는 작업을 지연시킬 수 있다.</li>
  <li>비동기 콜백이 브라우저에서 실행되면 web API, 콜백 큐, 이벤트 루프를 통해 관리된다.</li>
</div>

### ✅ 자바스크립트 가비지 컬렉션

<div class="blue-box">
  <li>가비지 컬렉션이란 더이상 사용되지 않는 메모리 영역을 해제하는 작업이다.</li>
  <li>우리가 사용하는 원시값, 객체, 함수 등은 모두 메모리를 차지하는데, 어떻게든 접근하거나 사용할 수 있는 값은 메모리에서 삭제하지 않는다.</li>
  <li>자바스크립트의 가비지 컬렉션은 <span class="highlight">mark-and-sweep 알고리즘</span>으로 동작한다.</li>
  <li>루트가 참조하고 있는 모든 객체를 방문하고 mark하는 과정을 반복하며, mark되지 않은 모든 객체를 메모리에서 삭제한다.</li>
</div>

## 📘 reference

- [MDN - Call_stack](https://developer.mozilla.org/ko/docs/Glossary/Call_stack)
- [면접 질문 깃헙 레포 - 콜스택과 힙](https://github.com/baeharam/Must-Know-About-Frontend/blob/main/Notes/javascript/stack-heap.md)
- [https://medium.com/sessionstack-blog/how-does-javascript-actually-work-part-1-b0bacc073cf](https://medium.com/sessionstack-blog/how-does-javascript-actually-work-part-1-b0bacc073cf)
- [https://ko.javascript.info/garbage-collection](https://ko.javascript.info/garbage-collection)
