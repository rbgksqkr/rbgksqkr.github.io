---
title: "[JS] 쓰로틀링 & 디바운스"
layout: single
excerpt: 쓰로틀링과 디바운스를 이해하며, 상황에 맞게 적절히 활용할 수 있다.

categories:
  - javascript
tags: [쓰로틀링, 디바운스, 이벤트]
toc_label: API 응답속도 비교
toc: true
toc_sticky: true
---

## 💭 예상 질문

- 디바운스와 쓰로틀링의 차이점은 무엇인가요?
  - 주로 어떤 상황에서 쓰로틀링과 디바운스를 사용하나요?
- React에서 쓰로틀링 또는 디바운스를 사용할 때 성능 최적화를 위해 어떤 점을 고려해야 하나요?
- useEffect 를 이용하여 쓰로틀링/디바운스을 구현할 때 주의해야 할 점은 무엇인가요?
- 쓰로틀링을 적용했을 때, 마지막 호출이 누락되는 문제를 어떻게 해결할 수 있을까요? (trailing)

<div class="red-box">
    <div>쓰로틀링이란 <mark class="mark">일정 시간 동안 여러 번 발생한 이벤트를 한 번만 실행되도록 하는 기술</mark>이고, 디바운스는 <mark class="mark">일정 시간 동안 이벤트가 다시 발생하지 않을 때까지 실행을 지연시키는 기술</mark>입니다.</div>
    <br />
    <div>쓰로틀링은 scroll이나 resize 처럼 이벤트가 자주 발생하는 경우에 사용하고, 디바운스는 사용자 입력을 멈췄을 때 API를 호출해야 하는 경우에 주로 사용합니다.</div>
    <br />
    <div>두 방식을 적절히 선택하여 <mark class="mark">과도한 이벤트 발생으로 인한 성능 저하를 방지</mark>할 수 있습니다.</div>
</div>

### ✅ 쓰로틀링

### ✅ 디바운스

## 📘 reference

- [MDN - Throttle](https://developer.mozilla.org/en-US/docs/Glossary/Throttle)
- [MDN - Debounce](https://developer.mozilla.org/en-US/docs/Glossary/Debounce)
