---
title: "[JS] 이벤트 위임"
layout: single
excerpt: 이벤트 위임을 이해하며, 모든 하위 요소에 이벤트 리스너를 달지 않고 문제를 해결할 수 있다.

categories:
  - javascript
tags: [이벤트, 버블링, 캡쳐링, 이벤트 위임]
toc_label: 이벤트 위임
toc: true
toc_sticky: true
---

## 💭 예상 질문

- 이벤트 위임이란 무엇이며, 왜 사용하는지 설명해주세요.
- 이벤트 위임이 DOM 구조 변경 시 어떻게 유용하게 작동하나요?
- 자바스크립트의 캡처링과 버블링 단계는 무엇이며, 이벤트 위임에 어떻게 영향을 미치나요?
- 버블링을 막기 위해 stopPropagation()을 사용하는 경우, 이벤트 위임에 어떤 영향을 줄까요?

<div class="red-box">
  <div>이벤트 위임이란 <mark class="mark">부모 요소에 이벤트 핸들러를 등록하여 자식 요소의 이벤트를 부모가 대신 처리하는 기법</mark>입니다.</div>
  <div>많은 자식 요소에 개별적으로 핸들러를 붙이는 대신, 부모에 한 번만 등록하여 <mark class="mark">메모리 사용을 줄일 수 있습니다.</mark></div>
  <div>자식 노드에 동일한 이벤트 동작이 발생하거나 동적으로 자식 노드가 추가되는 경우 사용합니다.</div>
</div>

## 📘 reference

- [https://ko.javascript.info/event-delegation](https://ko.javascript.info/event-delegation)
