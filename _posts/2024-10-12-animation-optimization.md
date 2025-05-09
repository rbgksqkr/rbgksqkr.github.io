---
title: "[브라우저 렌더링] 애니메이션 최적화를 통해 브라우저 렌더링 시간 94% 개선"
layout: single
excerpt: 구형 모바일 환경에서 frame drop이 발생하여 애니메이션이 끊기는 현상을 애니메이션 최적화를 통해 브라우저 렌더링 시간 94%를 개선하였다.

categories:
  - trouble-shooting
tags: [트러블 슈팅, 애니메이션 최적화, 브라우저 렌더링, 모바일, transform]
toc_label: 브라우저 렌더링 시간 94% 개선
toc: true
toc_sticky: true
---

PR 링크: [https://github.com/woowacourse-teams/2024-ddangkong/pull/297](https://github.com/woowacourse-teams/2024-ddangkong/pull/297)

## 🚨 문제 상황

중간 데모데이 당시, 부스를 운영하며 함께 땅콩 서비스를 이용하였는데, 사용자의 모바일 환경에서 `타이머가 심하게 끊기는 현상`을 발견하였습니다. 처음에는 현장 네트워크가 혼잡해 발생한 일시적인 지연이라 판단했습니다.

하지만 이후 듣게 된 성능 최적화 강의에서 소개된 사례가 우리 서비스 문제와 매우 유사했고, 우리가 놓친 성능 이슈일 수 있겠다는 생각에 직접 확인해보기로 하였습니다.

Chrome DevTools의 Performance 탭을 활용해 CPU 및 네트워크 쓰로틀링을 설정하고, 구형 모바일 환경을 재현해보았습니다. 그 결과, 로컬 환경에서도 프레임 드랍이 똑같이 재현되었고, 이는 <mark class="mark">일시적인 문제가 아닌 서비스 자체의 성능 문제임을 인지</mark>하게 되었습니다.

이에 팀원들과 이슈를 공유하고, 본격적인 성능 개선 작업을 진행하게 되었습니다.

## 문제 분석

CPU 쓰로틀링(x20)과 Slow 4G 네트워크 설정을 통해 구형 디바이스 환경을 재현하고 성능 측정을 해보니 문제를 알 수 있었습니다. 브라우저 렌더링 시간이 지연되고 있었고, 특히 레이아웃과 스타일 재계산, 페인트 비율이 높았습니다.

|                                                                                   개선전-요약                                                                                    |                                                                                     개선전-상향식                                                                                     |
| :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
| <img width="350" alt="게임화면-타이머_렌더링성능_개선전_요약" src="https://github.com/user-attachments/assets/4ff4aeac-89d8-480f-9ff7-e86289ea6ce3" /> | <img width="350" alt="게임화면-타이머_렌더링성능_개선전_상향식" src="https://github.com/user-attachments/assets/c6091869-53f0-4e5f-a3aa-81fccef3005e" /> |

성능 최적화 강의에서 width, right 등 요소의 기하학적인 요소를 변경하면 불필요한 reflow와 repaint가 발생한다는 것을 이해했습니다.

이는 기존의 타이머 애니메이션을 구현할 때 이미지는 right, 타이머 막대는 width를 변경하여 처리하고 있어서 해당 부분이 문제라는 것을 파악할 수 있었습니다. 따라서 <mark class="mark">reflow → repaint → composite 과정이 매 프레임마다 동작하여 렌더링 지연이 발생</mark>한 것입니다!

```ts
// 기존 코드
width: ${width}%;
right: ${100 - width}%;
transition: all 1s linear;
```

## ✅ 문제 해결

<div class="red-box">
    <div>문제는 <mark class="mark">렌더링 파이프라인을 너무 많이 돈다는 것</mark>이었습니다.</div>
</div>

<div class="blue-box">
    <div><b>레이아웃 변경을 발생시키지 않고 화면에 보이는 요소의 위치를 변경할 수 없을까?</b></div>
    <div>width, right 대신 <mark class="mark">transform 속성</mark>을 이용하면 reflow, repaint가 발생하지 않고 composite 단계만 거쳐 렌더링 파이프라인이 동작합니다.</div>
    <div>transform을 적용하면 브라우저가 해당 요소를 <mark class="mark">별도의 레이어로 분리하여 composite 단계만</mark> 거치며, <mark class="mark">GPU 가속을 활용</mark>하기 떄문에 브라우저가 빠르게 처리할 수 있습니다.</div>
</div>

```ts
// 타이머 막대
width: 100%;
transform: scaleX(${scale});
transform-origin: left;
transition: transform 1s linear;

// 타이머 이미지
transform: translateX(-${(1 - scale) * 98}%);
```

## 📘 요약

<div class="blue-box">
    <div>브라우저 렌더링 1686ms → 87ms, 약 94% 성능 개선</div>
</div>

렌더링 구조에 대한 이해 → 성능 문제 분석 → 실제로 체감되는 성능 개선 효과를 배울 수 있었습니다.

단순히 느려서 고쳤다가 아니라, 왜 느리게 동작했는지 이해하고, 브라우저의 동작 원리를 바탕으로 해결하여 성장할 수 있었습니다.
