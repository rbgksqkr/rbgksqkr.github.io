---
title: "[Web] transform 속성으로 애니메이션 최적화가 되는 이유"
layout: single
excerpt: transform 속성이 어떻게 애니메이션 최적화를 이뤄내며, 비대체 인라인 박스에는 왜 transform이 동작하는지 이해할 수 있다.

categories:
  - web
tags: [web, 브라우저 렌더링, 합성, transform, 애니메이션 최적화, will-change]
toc_label: CSS transform
toc: true
toc_sticky: true
---

> transform을 사용하면 reflow, repaint를 일으키지 않고 합성 단계만 거쳐 애니메이션 최적화를 통해 개선되었다.

<div class="red-box">
    <div><b>[ transform 속성을 활용하면 왜 합성 단계만 거치나요? ]</b></div>
    <div>transform은 좌표공간을 변형함으로써 문서 흐름을 방해하지 않고 콘텐츠의 형태와 위치를 바꾼다.</div>
    <div>transform을 적용하면 브라우저는 해당 요소를 <mark class="mark">별도의 레이어로 분리</mark>한다.</div>
    <div>레이어가 분리되면 해당 레이어의 변경 사항은 렌더 트리 전체에 영향을 주는 게 아니라, <mark class="mark">해당 레이어의 변환만을 적용</mark>한다.</div>
    <div>따라서 <mark class="mark">reflow, repaint 단계를 거치지 않고 합성 단계만 거친다.</mark></div>
    <br />
    <div><b>[ 합성 단계만 거치기 때문에 transform 속성이 빠른 건가요? ]</b></div>
    <div>CPU는 메인스레드 작업을 담당하는데, <mark class="mark">CPU가 무거운 연산을 처리 중이면 CPU 기반 애니메이션은 동작이 멈출 수 있다.</mark></div>
    <div>따라서 그래픽 처리에 최적화되어 있는 <mark class="mark">GPU를 활용하여 애니메이션을 병렬로 처리</mark>할 수 있다.</div>
    <div>transform을 사용하면 해당 요소를 별도의 레이어로 분리하면서 <mark class="mark">GPU 가속을 활용</mark>한다.</div>
    <br />
    <div>따라서, transform 속성은 별도의 레이어에서 처리되어 reflow를 발생시키지 않으며, GPU를 활용하여 병렬 처리되기 때문에 애니메이션 최적화가 이뤄진다.</div>
</div>

<img width="600" alt="image" src="https://github.com/user-attachments/assets/94eb6a85-d3f4-43c5-a499-50cad992d81f" />

## 비대체 인라인 박스, <colgroup>, <col> 은 transform을 적용할 수 없다.

`비대체 인라인 박스` : 인라인 박스에 속하면서 비대체 요소인 요소들이 생성하는 박스

- `<span>`, `<a>`, `<strong>`과 같은 인라인 요소들이 텍스트와 함께 생성하는 박스

Element는 **블록 박스**와 **인라인 박스**로 분류되며, **대체 요소**와 **비대체 요소**로도 분류된다.

> 경고: 변형 가능한 요소만 transform할 수 있습니다. 즉, CSS 박스 모델이 레이아웃을 결정하는 모든 요소 중 비대체 인라인 박스, 표 열 박스, 표 열 그룹 박스를 제외한 요소에만 적용할 수 있습니다. - MDN

## **왜 Non-Replaced Inline Boxes는 `transform`을 사용할 수 없나요?**

비대체 인라인 박스는 레이아웃이 <mark class="mark">텍스트의 흐름에 밀접하게 연결</mark>되어 있다.

이러한 요소에 transform이 적용되면 레이아웃을 재계산해야 한다.

colgroup과 col도 같은 이유로 transform이 적용되면 table의 레이아웃을 재계산해야 한다.

따라서 <mark class="mark">transform 속성을 사용하면 합성 단계만 이뤄지도록 타겟 요소에서 제외된걸로 이해</mark>할 수 있다.

---

### 테스트 결과

- 블록 박스인 div는 transform이 적용된다.
- 비대체 인라인 박스 요소인 span, a, strong 태그는 transform이 적용되지 않는다.
- colgroup과 col 태그도 transform이 적용되지 않는다.

```jsx
const TransformComponent = () => {
  return (
    <div>
      <span style={{ transform: 'translateX(100px) scale(1.2)' }}>span : 안녕하세요</span>
      <a style={{ transform: 'translateX(100px) scale(1.2)' }}>a : 안녕하세요</a>
      <strong style={{ transform: 'translateX(100px) scale(1.2)' }}>strong : 안녕하세요</strong>
      <div style={{ transform: 'translateX(100px) scale(1.2)' }}>div : 안녕하세요</div>
      <p style={{ transform: 'translateX(100px) scale(1.2) translateY(-20px)' }}>p : 안녕하세요</p>
      <table>
        <caption>transform 테스트</caption>
        <colgroup
          style={{ backgroundColor: 'lightgray', transform: 'translateX(100px) scale(1.2)' }}
        >
          <col />
          <col span={2} style={{ transform: 'translateX(100px) scale(1.2)' }} />
          <col span={2} style={{ transform: 'translateX(100px) scale(1.2)' }} />
        </colgroup>
        <tbody>
          <tr>
            <td></td>
            <th scope="col">Batman</th>
            <th scope="col">Robin</th>
            <th scope="col">The Flash</th>
            <th
              scope="col"
              style={{ backgroundColor: 'yellow', transform: 'translateX(10px) scale(1.2)' }}
            >
              Kid Flash
            </th>
          </tr>
          <tr>
            <th scope="row">Skill</th>
            <td>Smarts, strong</td>
            <td>Dex, acrobat</td>
            <td style={{ backgroundColor: 'green', transform: 'translateY(10px) scale(1.2)' }}>
              Super speed
            </td>
            <td>Super speed</td>
          </tr>
        </tbody>
      </table>
    </div>
  );
};

ReactDOM.createRoot(document.getElementById('root')!).render(<TransformComponent />);
```

<img width="500" alt="image" src="https://github.com/user-attachments/assets/a6e498c0-e537-4d70-b99b-a7a620e5b370">

## will-change

`will-change` : 브라우저에게 앞으로 변화를 예고하는 역할

특정 속성이 변경될 때 성능을 최적화할 수 있도록 컴포지터 레이어(compositor layer)를 미리 생성

transform 속성만 사용해도 새로운 레이어를 생성할 수 있지만, `will-change` 를 명시적으로 지정하면 브라우저가 해당 요소를 위한 새 레이어를 미리 준비한다.

`will-change` 는 브라우저가 메모리 내에 오랫동안 최적화 상태를 유지하게 하여 성능적인 문제가 생긴다. 과도한 최적화는 독이 되므로 성능 문제가 생겼을 때만 사용해야 한다.

## 📘 reference

- [렌더링 성능 - web.dev](https://web.dev/articles/rendering-performance?hl=ko)

- [CSS transform 사용 - MDN](https://developer.mozilla.org/ko/docs/Web/CSS/CSS_transforms/Using_CSS_transforms)

- [transform - MDN](https://developer.mozilla.org/ko/docs/Web/CSS/transform)

- [will-change - MDN](https://developer.mozilla.org/ko/docs/Web/CSS/will-change)

- [will-change의 숨겨진 의미 - 블로그](https://ghlee.dev/what-will-change-really-means)

- [CSS 왜 transform 애니메이션 성능이 좋을까 - 블로그](https://mong-blog.tistory.com/entry/CSS-%EC%99%9C-transform-%EC%95%A0%EB%8B%88%EB%A9%94%EC%9D%B4%EC%85%98-%EC%84%B1%EB%8A%A5%EC%9D%B4-%EC%A2%8B%EC%9D%84%EA%B9%8C-with-GPU-Reflow)
