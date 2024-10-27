---
title: "[크로스 브라우징] IOS 키보드 버그 개선"
layout: single
excerpt: 땅콩 서비스를 개발하면서 IOS 환경에서만 가상 키보드 위로 버튼이 올라오지 않는 문제를 해결하였다.

categories:
  - trouble-shooting
tags: [크로스브라우징, 우아한테크코스, 땅콩, IOS, 키보드]
toc_label: API 응답속도 비교
toc: true
toc_sticky: true
---

<div class="red-box">
  <div>모바일 환경에서 입력창을 포커스할 때, 키보드가 올라온다.</div>
  <div>화면 하단에 붙어있는 버튼을 키보드 위로 올리려고 한다.</div>
  <div>현재 구현한 방식은 android에서는 동작하는데, <mark class="mark">IOS에서 동작하지 않는다.</mark></div>
  <div>안되는 것도 아니고 왜 IOS에서만 안될까?</div>
</div>

## 🔍 현재 구현 방식

화면에 보이는 높이(`window.visualViewport.height`)를 초기값과 비교하여 키보드가 열렸는지를 판단하였다.

`document에 resize 이벤트를 등록` 하여 현재 보이는 화면의 높이가 기존 높이보다 낮아졌을 때 키보드가 올라온 것으로 판단하였다.

이렇게 구현하면 안드로이드에선 동작하지만, IOS에선 동작하지 않는다.

```ts
import { useEffect, useState } from "react";

const useKeyboardUp = () => {
  const [isKeyboardUp, setIsKeyboardUp] = useState(false);

  useEffect(() => {
    const initialHeight = window.visualViewport?.height;

    const handleResize = () => {
      const currentHeight = window.visualViewport?.height;
      if (initialHeight && currentHeight && currentHeight < initialHeight) {
        setIsKeyboardUp(true);
      } else {
        setIsKeyboardUp(false);
      }
    };

    document.addEventListener("resize", handleResize);

    return () => {
      document.removeEventListener("resize", handleResize);
    };
  }, []);

  return { isKeyboardUp };
};
export default useKeyboardUp;
```

## 🚨 문제점

키보드에 의해 화면이 올라갈 때 android와 IOS에서 화면을 처리하는 방식이 달랐다.

화면이 올라갈 때 android에서는 window에 걸린 resize 이벤트가 발생하고, IOS에서는 이벤트가 발생하지 않아 제대로 동작하지 않았다.

IOS에서 document에 resize 이벤트가 발생하지 않은 이유는 <mark class="mark">키보드에 의해 화면이 올라갈 때 viewport를 조절하는 것이 아니고,</mark> <span class="high">document 자체를 키보드 높이만큼 밀어버린다고 한다.</span>

<img width="500" alt="IOS 키보드 동작 방식" src="https://github.com/user-attachments/assets/c94353d8-61f0-4045-81b2-d1ba5689efa9">

개선해야할 부분은 크게 4가지가 있었고, 각각에 대한 해결 방안은 아래에 별도로 작성하였다.

- 가상 키보드로 인해 resize 이벤트가 동작하지 않아 버튼이 안올라오는 현상
- 입력창을 클릭했을 때 확대되는 현상
- IOS 자체적으로 인풋창이 화면의 가운데에 위치시키기 위해 스크롤을 발생시키는 현상
- 가상 키보드가 올라오면서 그 공간으로 스크롤이 생기는 현상

## ❌ 개선 전

### 가상 키보드로 인해 resize 이벤트가 동작하지 않아 버튼이 안올라오는 현상

- 안드로이드는 viewport가 줄어들지만, IOS는 viewport는 그대로고 document 가 키보드 높이만큼 올라가 resize 이벤트가 발생하지 않았다.
- 이를 <mark class="mark">document 대신 visualViewport에 resize 이벤트를 걸어 해결</mark>하였다.

### 입력창을 클릭했을 때 확대되는 현상

- 접근성 정책으로 인해 font-size가 16px 이하인 경우 브라우저에서 입력창을 확대시킨다고 한다. [(stack overflow)](https://stackoverflow.com/questions/2989263/disable-auto-zoom-in-input-text-tag-safari-on-iphone/6394497#6394497)
- meta 태그에서 줌을 막을 수 있지만 이것은 웹접근성에 악영향을 미친다고 한다. (user scalable)
- 폰트 크기가 커졌을 때 UI가 크게 이상해지지 않고, 오히려 사용성을 높인다 생각하여 <mark class="mark">font-size를 16px로 키워 해결</mark>하였다.

### IOS에서 인풋창을 화면의 가운데에 위치시키기 위해 스크롤을 발생시키는 현상

- IOS 자체적으로 키보드가 인풋을 가리지 않기 위해 인풋을 가운데로 위치시키는 스크롤이 발생한다.
- IOS 버그(?)라고 판단하여 <mark class="mark">헤더 높이 조정 & 불필요한 여백 제거로 간격을 줄여 해결</mark>하였다.

<div style="display:flex; justify-content:center; gap: 16px">
<img width="250" alt="image" src="https://github.com/user-attachments/assets/ab1f0eeb-9055-4d6b-97bd-c97615976f91"> <img width="250" alt="image" src="https://github.com/user-attachments/assets/6a00223a-3099-4064-879b-8d045486aa58">
</div>

### 가상 키보드가 올라오면서 그 공간으로 스크롤이 생기는 현상

여러 방식을 사용했지만 [해당 블로그](https://im-developer.tistory.com/201)에서 소개하는 가장 쉬우면서도 현재 상황에 적절한 <mark class="mark"><span class="high">touchmove 이벤트</span> 동작을 막아 스크롤이 되지 않도록 설정하여 해결</mark>하였다.

[https://github.com/user-attachments/assets/209c1f85-0974-4eac-9942-714064004982](https://github.com/user-attachments/assets/209c1f85-0974-4eac-9942-714064004982)

## ✅ 개선 후 (코드 포함)

[https://github.com/user-attachments/assets/fb991e58-45c6-4716-add7-1bad69bea282](https://github.com/user-attachments/assets/fb991e58-45c6-4716-add7-1bad69bea282)

```ts
import { useEffect, useState } from "react";

const INITIAL_BOTTOM_BUTTON_HEIGHT = 0;

const useButtonHeightOnKeyboard = () => {
  const [bottomButtonHeight, setBottomButtonHeight] = useState(
    INITIAL_BOTTOM_BUTTON_HEIGHT
  );

  useEffect(() => {
    const handleLockScroll = (e: TouchEvent) => {
      e.preventDefault();
    };

    document.body.addEventListener("touchmove", handleLockScroll, {
      passive: false,
    });
    return () => {
      document.body.removeEventListener("touchmove", handleLockScroll);
    };
  }, []);

  useEffect(() => {
    const handleResizeScreen = () => {
      if (!visualViewport) return;

      setBottomButtonHeight(window.innerHeight - visualViewport.height);
    };

    visualViewport?.addEventListener("resize", handleResizeScreen);

    return () => {
      visualViewport?.removeEventListener("resize", handleResizeScreen);
    };
  }, []);

  return { bottomButtonHeight };
};

export default useButtonHeightOnKeyboard;
```

## 🚨 화면 높이 계산

#### innerHeight

- 수평 스크롤 막대 높이를 포함한 화면 높이
- window의 layout viewport의 높이에서 가져옴

> The read-only innerHeight property of the Window interface returns the interior height of the window in pixels, including the height of the horizontal scroll bar, if present.
>
> The value of innerHeight is taken from the height of the window's layout viewport. The width can be obtained using the innerWidth property. - MDN

#### clientHeight

- 스크롤바가 차지하는 영역을 제외한 화면 높이
- padding을 포함하지만, 수평 스크롤바의 높이, border, margin은 포함 ❌
- document.documentElement.clientHeight = height + padding - 수평 스크롤바의 높이(존재하는 경우에만)로 계산됨

#### visualViewport.height

- 주어진 창에 대한 시각적 viewport 높이

### 📘 reference

- [iOS15 대응기 (feat. 크로스 브라우징) - 채널톡](https://channel.io/ko/blog/articles/12bccbc3)
- [https://velog.io/@gene028/ios-keyboard](https://velog.io/@gene028/ios-keyboard)
- [https://im-developer.tistory.com/201](https://im-developer.tistory.com/201)
