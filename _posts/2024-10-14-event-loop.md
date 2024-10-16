---
title: "[JS] 이벤트 루프"
layout: single
excerpt: 이벤트 루프를 이해하며, 자바스크립트가 어떻게 비동기 처리하는지 이해할 수 있다.

categories:
  - javascript
tags: [비동기]
toc_label: 이벤트 루프
toc: true
toc_sticky: true
---

## 💭 예상 질문

- 이벤트 루프에 대해 설명해주세요.
- 자바스크립트는 싱글 스레드 언어인데 비동기 작업은 어떻게 처리되나요?
- setTimeout과 Promise는 이벤트 루프에서 어떻게 처리되나요?
- `requestAnimationFrame` 은 이벤트 루프에서 어떻게 동작하나요?
- 비동기 처리와 이벤트 루프에 대한 성능 최적화 방법은 무엇이 있나요?

<div class="red-box">
    <div>이벤트 루프는 비동기 작업을 처리하는 메커니즘으로, <mark class="mark">태스크 큐에 있는 비동기 작업을 우선순위에 따라 콜스택에 넣어 실행</mark>시킵니다.</div>
    <div>자바스크립트를 읽다가 비동기 작업을 만나면 태스크 큐에 넣고,<mark class="mark">콜스택이 비었을 때 이벤트 루프가 동작하여 태스크큐에 있는 비동기 작업을 실행</mark>합니다.</div>
    <br />
    <div>태스크 큐에는 <mark class="mark">macro task queue</mark>, <mark class="mark">micro task queue</mark>, <mark class="mark">animation frame</mark>로 3가지가 존재하며, 우선순위에 따라 처리됩니다.</div>
    <div style="font-size: 80%">*작업 우선순위 : micro task queue > animation frame > macro task queue</div>
</div>

## 📘 reference

- [MDN - event loop](https://developer.mozilla.org/ko/docs/Web/JavaScript/Event_loop)
- [https://ko.javascript.info/event-loop](https://ko.javascript.info/event-loop)
