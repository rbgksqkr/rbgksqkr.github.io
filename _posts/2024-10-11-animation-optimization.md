---
title: "[브라우저 렌더링] 모바일 환경에서 발생한 애니메이션 버벅임 현상 해결"
layout: single
excerpt: 모바일 환경에서 발생한 애니메이션 버벅임 현상에 대한 문제 상황과 해결 방법에 대해 소개한다.

categories:
  - trouble-shooting
tags: [트러블 슈팅, 애니메이션 최적화, 브라우저 렌더링, 모바일]
toc_label: 모바일 애니메이션 버벅임 현상 해결
toc: true
toc_sticky: true
---

<div class="red-box">
    <div>모바일 크롬과 사파리에서 애니메이션 버벅임이 발생하였다.</div>
    <div>PC에서는 발생하지 않아 몰랐는데, 다른 팀원들도 <mark class="mark">모바일 환경에서 애니메이션이 버벅이는 문제를 확인</mark>하였다.</div>
    <div>왜 모바일에서만 문제가 발생했을까?</div>
</div>

## 🔍 현재 구현 방식

<div class="box">
    <div>scale 값을 state로 관리하고, 게임 라운드의 제한 시간인 timeLimit으로 scale 값을 매초 계산하였다.</div>
    <div><mark class="mark">감소 비율을 (1 / timeLimit) 로 계산하고 transform에 주입하여 구현</mark>하였다.</div>
    <div>나누기 연산으로 매 프레임마다 연산이 들어가면서 애니메이션이 매끄럽지 않게 동작하였다.</div>
</div>

```jsx
export const timerInnerLayout = (scale: number) => css`
  transform: scaleX(${scale});
  transform-origin: left;
  transition: transform 1s linear;
`;

export const timerWrapper = (scale: number) => css`
  transform: translateX(-${(1 - scale) * 98}%);
  transition: transform 1s linear;
`;
```

## 🚨 문제점

<div class="box">
    <div>scaleX의 인자는 숫자(비율)가 들어가는데, 타이머 막대가 있었다가 없어져야 하므로, 1에서 0으로 줄어들도록 구현하였다.</div>
    <div>게임 시간이 방설정에 따라 달라지므로 타이머가 줄어드는 비율을 계산하여 인자로 넘겨줬다.</div>
    <div>타이머 내부 막대가 scaleX로 줄어들면서 땅콩 타이머 이미지가 translateX로 같이 움직이는데, 계산한 비율만큼 이동하는 로직에서 문제가 발생한다.</div>
    <div>나누기가 딱 떨어지지 않을 경우, <mark class="mark">나눈 비율로 transform한 거리와 실제 화면상에서 이동한 거리가 달라 문제가 생긴 것으로 확인</mark>되었다.</div>
</div>

## ✅ 문제 해결

<div class="box">
    <div>모바일 환경은 PC에 비해 성능도 느리고, 비율 계산이 정확하지 않아 발생한 것으로 판단하였다.</div>
    <div>기존에 scale값을 계산하기 위해 처리한 나누기 연산을 제거하고, timeLimit 만큼 애니메이션 수행한다. 이전보다 애니메이션 재계산이 줄어 훨씬 매끄럽게 동작한다.</div>
</div>

```jsx
const progress = keyframes`
  0% {
      transform: scaleX(1);
  }
  100% {
      transform: scaleX(0);
    }
`;

const timerTransition = keyframes`
  0% {
      transform: translateX(0);
  }
  100%{
      transform: translateX(-95%);
  }
`;

export const timerInnerLayout = (timeLimit: number) => css`
  transform-origin: left;
  animation: ${progress} ${timeLimit + 1}s linear;
`;

export const timerWrapper = (timeLimit: number) => css`
  animation: ${timerTransition} ${timeLimit + 1}s linear;
`;
```

|                                                             애니메이션 개선 전                                                             |                                                            애니메이션 개선 후                                                             |
| :----------------------------------------------------------------------------------------------------------------------------------------: | :---------------------------------------------------------------------------------------------------------------------------------------: |
| <img width="350" alt="애니메이션_개선전" src="https://github.com/user-attachments/assets/3c71bfc8-e44d-4b98-a8a7-6c4d6f501cd3"> | <img width="350" alt="애니메이션_개선후" src="https://github.com/user-attachments/assets/dc6f5990-a766-4478-a34f-53b9cb216dec"> |

## 💡 결론

<div class="blue-box">
    <div>나누기 연산으로 인해 애니메이션이 프레임마다 재계산되면서 끊기는 문제가 발생한다.</div>
    <div>또한 모바일 환경은 PC에 비해 느리고 <span class="high">부동소수점 계산 오차</span>로 비율 계산이 정확하지 않아, <mark class="mark">계산한 값와 실제 화면에 그려지는 위치 간에 차이가 발생</mark>한다.</div>
    <div>나누기 연산을 제거하고 <mark class="mark">timeLimit 동안 애니메이션을 동작시켜 부드럽게 실행</mark>한다.</div>
</div>
