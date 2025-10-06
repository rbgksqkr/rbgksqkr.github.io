---
title: "[반응형] grid-template-areas로 반응형 UI 유연하게 대응하기"
layout: single
excerpt: PC와 모바일 UI가 달라도 하나의 컴포넌트로 관리하며, grid-template-areas를 활용해 레이아웃을 유연하게 대응한 경험을 소개한다.
categories:
  - trouble-shooting
tags: [CSS Grid, 반응형, grid-template-areas]
toc_label: grid-template-areas
toc: true
toc_sticky: true
---

<div class="red-box">
  <div>
    PC와 모바일의 UI 레이아웃이 다르다는 이유로 <mark class="mark">조건부 렌더링</mark>과 <mark class="mark">display: none</mark>을 반복적으로 사용하고 있었습니다.
  </div>
  <div>
    이로 인해 동일한 UI 요소를 두 번 렌더링해야 했고, 관리가 어렵고 비효율적인 코드가 되어버렸습니다.
  </div>
  <div>
    이번 리팩토링을 통해 <mark class="mark">grid-template-areas</mark>만으로 하나의 DOM으로 PC와 모바일 UI를 모두 대응하여 수정하기 쉬운 코드로 개선되었습니다.
  </div>
</div>

---

## 🔥 결론

조건부 렌더링이나 display:none을 남발하지 않아도 grid-area, grid-template-areas 로 반응형 UI를 유연하게 대응할 수 있었습니다.

|                                                                            PC 홈화면 분석                                                                             |                                                                          Mobile 홈화면 분석                                                                           |
| :-------------------------------------------------------------------------------------------------------------------------------------------------------------------: | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
| <img width="825" height="354" alt="스크린샷 2025-10-06 오후 7 31 59" src="https://github.com/user-attachments/assets/2b8a5ddd-8fbe-403e-84e0-3e9e9767ad8c" /> | <img width="443" height="371" alt="스크린샷 2025-10-06 오후 7 31 51" src="https://github.com/user-attachments/assets/75038912-8c3c-4066-976c-4cf279052105" /> |

|                                                            PC 카드 분석                                                            |                                                          Mobile 카드 분석                                                          |
| :--------------------------------------------------------------------------------------------------------------------------------: | :--------------------------------------------------------------------------------------------------------------------------------: |
| <img width="840" height="294" alt="image" src="https://github.com/user-attachments/assets/e493bab7-a1b9-4cd9-a3f6-65a01396a241" /> | <img width="840" height="251" alt="image" src="https://github.com/user-attachments/assets/8779e832-f992-462e-9f8f-1f2eaeedf7ca" /> |

## 🚨 문제 상황

홈화면의 전체적인 레이아웃을 비교하면 제목, 필터, 체크박스, 전체 공고 수의 위치가 PC와 Mobile에서 다릅니다.

이를 대응하기 위해 컴포넌트를 2개로 만들거나 HTML 태그를 2개 만들어서 display: none을 viewport에 따라 적용해야 했습니다.

|                                              PC 홈화면 디자인                                              |                                            Mobile 홈화면 디자인                                             |
| :--------------------------------------------------------------------------------------------------------: | :---------------------------------------------------------------------------------------------------------: |
| <img  alt="image" src="https://github.com/user-attachments/assets/a7d58742-dcb4-4f1f-a9f4-e56f07cdd73a" /> | <img   alt="image" src="https://github.com/user-attachments/assets/03c70224-770b-4102-ac24-a099905bccf2" /> |

|                                              PC 카드 디자인                                               |                                             Mobile 카드 디자인                                             |
| :-------------------------------------------------------------------------------------------------------: | :--------------------------------------------------------------------------------------------------------: |
| <img alt="image" src="https://github.com/user-attachments/assets/23252a60-fd56-4be6-aa3a-f46a3bd8f012" /> | <img  alt="image" src="https://github.com/user-attachments/assets/a68ff6e6-f876-46bf-915a-c3d1dfe89043" /> |

이전에는 이런 레이아웃 구조 차이를 해결하기 위해 아래와 같이 코드를 작성했습니다.

media-query 를 활용하여 viewport 너비에 따라 `display: none` 을 적용하여 HTML 태그를 두 벌로 관리했습니다.

아래와 같이 PC 화면에서 보이는 태그와 Mobile 화면에서 보이는 태그가 2개씩 존재했어요.

```tsx
// Component.css.ts
export const postViewWrapperDesktop = style({
  display: "flex",
  alignItems: "center",
  gap: "0.4rem",

  "@media": {
    "screen and (max-width: 767px)": {
      display: "none",
    },
  },
});

export const postViewWrapperMobile = style({
  gridArea: "views",
  display: "none",

  "@media": {
    "screen and (max-width: 767px)": {
      display: "flex",
      alignItems: "center",
      gap: "0.4rem",
    },
  },
});

// Component.tsx
const Component = () => {
  return (
    <>
      <div className={postViewWrapperDesktop}>
        <Icon icon="Eye" width={18} height={18} color={colors.icon02} />
        <span className={postViews}>{views}</span>
      </div>
      ...
      <div className={postViewWrapperMobile}>
        <Icon icon="Eye" width={18} height={18} color={colors.icon02} />
        <span className={postViews}>{views}</span>
      </div>
    </>
  );
};
```

이 방식은 작동은 했지만, display:none으로 숨기는 코드가 증가하면서 유지보수하기 어려운 코드가 되어가고 있었어요. Context를 알아야만 코드를 수정할 수 있다면 수정하기 어려운 코드이고, 이는 구현이 복잡하게 되어있는 코드라고 생각해요.

> UI 구조는 하나로 유지하고, 배치만 다르게 가져갈 수는 없을까? -> `grid-template-areas`

### ✅ 문제 해결

아래는 실제로 적용한 grid-template-areas 코드의 요약 버전입니다.

이렇게 적용하니 기존의 `display: none`으로 처리하여 불필요하게 2벌로 관리되던 중복 코드를 제거할 수 있었고, 유지보수 관점에서도 체크박스 1개를 수정하면 웹/모바일 동일하게 적용되니 수정하기 쉬워졌어요.

주의할 점은 <mark class="mark">특정 행 또는 열만 다른 간격을 주는 게 번거로웠어요</mark>. 일부만 gap을 수정하기 위해 <b>음수 margin</b>을 사용했다가 추가 대응이 어려워질 것 같아 `gap을 모두 지정해주는 방식`을 선택했는데 적절한 판단이였다고 생각해요.

```ts
export const postContainerLayout = style({
  display: "grid",
  gridTemplateRows: "auto 1.6rem auto 1.6rem auto 0.6rem auto",
  gridTemplateAreas: `
    "title title"
    ". ."
    "filters checkbox"
    ". ."
    "count count"
    ". ."
    "posts posts"
  `,

  "@media": {
    "screen and (max-width: 767px)": {
      gridTemplateRows: "auto 0rem auto 0rem auto 0.6rem auto",
      gridTemplateAreas: `
        "divider divider"
        ". ."
        "filters filters"
        ". ."
        "count checkbox"
        ". ."
        "posts posts"
      `,
    },
  },
});

export const postContainerTitleDesktop = style({
  gridArea: "title",
});

export const horizontalLineMobile = style({
  gridArea: "divider",
});

export const filterWrapper = style({
  gridArea: "filters",
});

export const recruitCheckWrapper = style({
  gridArea: "checkbox",
});

export const totalPostCountWrapper = style({
  gridArea: "count",
});

export const postListContainer = style({
  gridArea: "posts",
});
```

### 🥕 효과

구현은 단순하지만, 이런 기능이 있는지 모른다면 디자인을 다시 요청드리거나 노가다로 구현하는 방향을 잡을 것 같아요. 웹/모바일 환경에 따라 조건부 렌더링하는 로직이 들어가거나 display:none이 남발되어 있는 코드를 본다면 수정하기 어려워질 거라고 생각해요.

이러한 문제점들을 미리 파악하고 대응했던 경험이라고 생각해요.

자바스크립트를 안쓰고 css로만 활용해서 구현하여 최선의 성능을 내고, 수정하기 쉬운 코드 구조를 만들 수 있도록 노력하는 방향이 의미 있었다고 생각해요.

처음에 개념을 알고도 레이아웃 구조를 맞춰서 바꾸는 게 어렵다고 생각했는데, 변경되지 않는 요소와 화면 너비에 따라 위치가 달라지는 요소를 명확히 구분해 grid-area를 설계하니 유연하게 대응할 수 있었습니다.

### 📘 Reference

- [https://github.com/YAPP-Github/25th-Web-Team-2-FE/pull/247](https://github.com/YAPP-Github/25th-Web-Team-2-FE/pull/247)
- [MDN – grid-template-areas](https://developer.mozilla.org/en-US/docs/Web/CSS/grid-template-areas)
- [MDN – grid-area](https://developer.mozilla.org/en-US/docs/Web/CSS/grid-area)
