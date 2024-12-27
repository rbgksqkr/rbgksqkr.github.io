---
title: "[Web] 브라우저 렌더링 (1) - 브라우저 구성 요소"
layout: single
excerpt: 브라우저에 대해 이해하며, 어떻게 화면에 출력하는지 이해할 수 있다.

categories:
  - web
tags: [web, 브라우저 렌더링, 브라우저 구성 요소]
toc_label: 브라우저 렌더링 (1) - 브라우저 구성 요소
toc: true
toc_sticky: true
---

<div class="red-box">
  <div>우리는 당연한 것처럼 개발을 시작할 때 localhost를 띄우고, React로 개발을 시작한다.</div>
  <div>그러다 갑자기 궁금증이 생겼다.</div>
  <div>브라우저는 어떻게 파일을 가져와서 화면에 띄우는거지? 파일을 어떻게 해석하는거지?</div>
  <div>Chrome 개발 문서를 기반으로 이해하며, 코드로 직접 구현해보자.</div>
</div>

<div class="blue-box">
  <div>1. 브라우저의 기능과 역할, 구성 요소에 대해 이해하자.</div>
</div>

## 📘 브라우저의 기본 기능

- 브라우저의 역할: 리소스를 서버에 요청하고, 응답받은 리소스를 화면에 출력
- 리소스의 종류는 HTML, PDF, 이미지 등이 있으며, 리소스 위치는 `URI`를 통해 지정
- 브라우저가 HTML 파일을 해석하고 표시하는 방식은 HTML 및 CSS 사양에 지정되어 있으며, 이는 W3C에서 관리
- 일반적인 브라우저 UI: 주소 표시줄, 뒤로가기/앞으로가기 버튼, 북마크, 새로고침 버튼, 중지 버튼, 홈 버튼

### ✅ Chrome 브라우저 특징

- 브라우저 메인 프로세스가 여러 탭을 관리한다.
- 각 탭은 <mark class="mark">별도의 프로세스에서 실행</mark>되며, 렌더링 엔진의 인스턴스를 탭마다 하나씩 실행한다.
  - 멀티 프로세싱 이유: 탭 1개가 무응답 상태가 되더라도 다른 탭에 영향을 주지 않는다.

## 📘 브라우저 구성 요소

1. **사용자 인터페이스**: 요청된 페이지가 표시되는 창을 제외한 브라우저 디스플레이의 모든 부분
2. **브라우저 엔진**: UI와 렌더링 엔진 간의 작업 마샬링(데이터 가공)
3. **렌더링 엔진**: 요청된 콘텐츠를 화면에 출력하는 역할
   - ex) 요청된 콘텐츠가 HTML인 경우, 렌더링 엔진은 HTML과 CSS를 파싱하고 파싱된 콘텐츠를 화면에 표시
4. **네트워킹**: HTTP/HTTPS 요청 같은 네트워크 작업 처리 (플랫폼 독립적)
5. **UI 백엔드**: 플랫폼과 독립적인 공통 인터페이스를 제공하여 UI 요소를 그림. 그래픽 라이브러리 내장.
6. **JavaScript 인터프리터:**: 자바스크립트 코드를 파싱하고 실행
7. **데이터 스토리지:** 쿠키와 같은 모든 종류의 데이터를 로컬에 저장 가능
   - ex) localStorage, IndexedDB 등 저장소 메커니즘 지원

<img width="400" src="https://github.com/user-attachments/assets/e3a174a8-06ed-4f45-92bd-6dabb997fa7c" />

## 📘 렌더링 엔진

브라우저의 렌더링 엔진은 `요청된 콘텐츠를 화면에 출력하는 역할` 을 한다.

요청된 콘텐츠가 HTML인 경우 렌더링 엔진은 <mark class="mark">HTML과 CSS를 파싱하고 파싱된 콘텐츠를 화면에 표시</mark>한다.

Safari는 `WebKit`, Chrome은 `Blink` 사용 (WebKit 기반)

<div class="blue-box">
  <div>1. HTML을 파싱하여 DOM 트리를 생성한다.</div>
  <div>2. CSS 파일과 스타일 요소에서 스타일 데이터를 파싱하여 CSSOM 트리를 생성한다.</div>
  <div>3. DOM 트리와 CSSOM 트리를 결합하여 화면에 출력할 렌더 트리를 생성한다.</div>
  <div>4. 각 노드가 화면에 표시될 위치를 계산하기 위해 레이아웃 단계를 거친다.</div>
  <div>5. 렌더 트리를 탐색하며 각 노드가 UI 백엔드 레이어를 사용하여 페인팅 단계를 거친다.</div>
  <div>6. 레이아웃 단계에서 생성한 레이어별로 페인트를 하고, 합성 단계에서 레이어를 합성하여 최종 화면을 출력한다.</div>
</div>

<img width="500" src="https://github.com/user-attachments/assets/1c9aaea8-c132-4dec-af89-5f797c30c1f2" />

---

네트워킹 레이어에서 요청된 문서의 콘텐츠를 가져오기 시작하며, 일반적으로 <mark class="mark">8KB 청크 단위</mark>로 이루어진다.

이 과정은 `점진적으로 진행`되며, 렌더링 엔진은 최대한 빨리 콘텐츠를 화면에 표시하려고 시도한다.

렌더링 트리의 빌드 및 레이아웃을 시작하기 전에 <mark class="mark">모든 HTML이 파싱될 때까지 기다리지 않는다.</mark>

콘텐츠의 일부가 파싱되고 표시되는 동안 프로세스는 네트워크에서 계속 수신되는 나머지 콘텐츠로 계속 진행된다.

아래는 WebKit의 세부적인 동작 흐름이다.

<img width="500" src="https://github.com/user-attachments/assets/42eda753-0bee-4cf2-b771-72823e223e0d" />

## 📘 reference

- [how browsers work? - Chrome dev](https://web.dev/articles/howbrowserswork?hl=ko)
