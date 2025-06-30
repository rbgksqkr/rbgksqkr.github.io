---
title: "[리액트 렌더링] 저사양 환경에서 입력 지연이 발생하여 useTransition으로 상태 업데이트 우선순위를 조정하여 INP 17%개선"
layout: single
excerpt: useTransition을 활용하여 입력 지연을 개선한 경험을 소개한다.

categories:
  - trouble-shooting
tags: [리액트 렌더링 최적화, useTransition, INP]
toc_label: 렌더링 최적화
toc: true
toc_sticky: true
---

<div class="red-box">
  <div>컴포넌트를 분리하는 것 만으로, RHF의 구독시스템을 이해하니 리렌더링이 많이 개선되었습니다.</div>
  <div>하지만 trigger를 입력할 때마다 호출해서 그런지, 여전히 살짝 느린 부분이 있었습니다.</div>
  <div>trigger의 동작과 입력 지연이 발생하는 이유를 좀더 자세히 알아보겠습니다.</div>
</div>

## 🔥 결론

2차 개선에는 <mark class="mark">useTransition을 활용하여 유효성 검증의 우선순위를 낮춰</mark> 사용자 입력을 최대한 방해하지 않도록 처리하였습니다.

|          -           |                                                             1차 개선                                                             |                                                           2차 개선                                                           |
| :------------------: | :------------------------------------------------------------------------------------------------------------------------------: | :--------------------------------------------------------------------------------------------------------------------------: |
| INP<br/>(136 -> 112) | <img width="400" alt="1차_개선_INP" src="https://github.com/user-attachments/assets/09094a16-c981-47f4-92ae-c17cbfad68e8" /> | <img width="400" alt="2차_개선_INP" src="https://github.com/user-attachments/assets/95d40169-f3c6-4c8d-b92e-0f0cfb1f9834" /> |

그 결과 스크립트 실행 시간 `약 69%`, INP `약 33%`, input 이벤트 당 실행 시간 `약 49%` 를 개선할 수 있었습니다.

|     |             -             |     |     | 개선 전 |     |     | 1차 개선 |     |     | 2차 개선           |     |
| :-: | :-----------------------: | :-: | :-: | ------- | --- | --- | -------- | --- | --- | ------------------ | --- |
|     |            INP            |     |     | 168ms   |     |     | 136ms    |     |     | **112ms (약 33%)** |     |
|     | input 이벤트 당 실행 시간 |     |     | 19.4ms  |     |     | 13ms     |     |     | **9.8ms (약 49%)** |     |

## 문제 상황

### 입력 지연이 발생하는 이유

입력 지연이란 사용자가 입력했을 때, UI에 바로 반영되지 않는 현상입니다. 왜 바로 반영되지 않을까요?

제어 컴포넌트를 사용하고 있으니 UI에 인풋이벤트를 반영하는 건 setState입니다. 반영되지 않는 건 <mark class="mark">onChange의 setState가 늦게 수행되기 때문</mark>이라고 생각해볼 수 있는데요.

setState를 수행하느라 <mark class="mark">메인 스레드가 바쁘다면, 브라우저가 이벤트를 전달하여 콜백을 실행시킬 수 없어서 사용자 입력 이벤트가 대기</mark>하게 됩니다.

> 사용자 입력 이벤트가 대기하지 않고 바로 반영되려면 어떻게 해야할까요?

## 문제 해결

메인 스레드에서 수행하고 있는 작업이 메인 스레드를 양보하고 사용자 입력 이벤트를 전달받으면 문제를 해결할 수 있습니다.

React 18에는 스케줄러가 존재하는데요. <mark class="mark">useTransition을 활용하면 특정 작업의 우선순위를 낮춰서 처리</mark>할 수 있습니다.

저는 onChange 이벤트 핸들러에서 trigger를 호출하고 있고, trigger로 인해 formState가 업데이트되고 리렌더링이 발생합니다. 저희는 formState 업데이트보다 사용자 입력이 UX에 중요하다고 판단했기 때문에 이를 적용해볼 수 있었습니다.

```tsx
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  setState(e.target.value);

  startTransition(() => {
    trigger(name);
  }
}
```

<mark class="mark">사용자 입력이 반영되는 setState는 우선순위를 유지</mark>하고, <mark class="mark">trigger를 startTransition으로 감싸 반응성을 개선</mark>할 수 있었습니다.

useTransition을 사용하면 우선순위에 따라 특정 작업이 지연될 수 있는데, 사용자 입력과 같은 더 높은 우선순위 작업에 의해 중단될 경우, 해당 Transition Lane의 작업을 버리고 최신값으로 재계산합니다.

이로 인해 빠르게 입력할수록 입력 이벤트로 인한 리렌더링 작업이 중단되어, <mark class="mark">input 이벤트 당 스크립트 실행 시간이 감소</mark>하는 것을 확인할 수 있습니다.

하지만 빈번한 재스케줄링으로 인해 총 스크립트 실행 시간은 증가할 수 있다는 점은 주의해야 합니다.

### Scheduler는 어떻게 렌더링을 중단하는걸까?

web API의 `isInputPending` 을 호출하여 대기 중인 input 이벤트를 확인합니다.

```ts
const isInputPending =
  typeof navigator !== "undefined" &&
  navigator.scheduling !== undefined &&
  navigator.scheduling.isInputPending !== undefined
    ? navigator.scheduling.isInputPending.bind(navigator.scheduling)
    : null;
```

하지만 safari, firefox 등은 해당 web API를 지원하지 않아 폴백을 사용하는데, <mark class="mark">5ms 이상 지연될 때 고정으로 콜스택을 비운다</mark>고 합니다.

<img width="700" alt="image" src="https://github.com/user-attachments/assets/34f7a7ca-c0bb-4800-9c17-8565a6277c1b" />

따라서, Concurrent Lane으로 작업 중일 때 대기 중인 input 이벤트가 있거나 5ms 이상 지연되면, 콜스택을 비워 macrotask 내의 이벤트 핸들러가 실행되도록 처리합니다.

<mark class="mark">React가 이벤트 루프에 관여하는 게 아니라, 콜스택을 비워 브라우저의 이벤트 루프가 동작하여 대기 중인 큐를 소비하는 방식</mark>을 활용합니다.

## 📘 reference

- [isInputPending - MDN](https://developer.mozilla.org/en-US/docs/Web/API/Scheduling/isInputPending)
- [Scheduler.js - v18.3.1](https://github.com/facebook/react/blob/f1338f8080abd1386454a10bbf93d67bfe37ce85/packages/scheduler/src/forks/Scheduler.js#L97)
