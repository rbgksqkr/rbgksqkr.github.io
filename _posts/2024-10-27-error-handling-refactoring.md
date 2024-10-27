---
title: "[에러 핸들링] 에러 처리 공통 로직 분리"
layout: single
excerpt: tanstack-query 를 활용하여 에러 처리 공통 로직을 분리하고, 이때 생긴 문제 상황과 해결 방법을 공유한다.

categories:
  - trouble-shooting
tags:
  [땅콩, 트러블 슈팅, 에러 핸들링, Context API, QueryCache, setDefaultOptions]
toc_label: 에러 처리 공통 로직 분리
toc: true
toc_sticky: true
---

<div class="red-box">
  <div>땅콩 서비스에서는 서버 상태를 tanstack-query로 관리하며, mutation 에는 모달로 에러 핸들링을 하고 있다. 그런데 같은 에러 처리인데 mutation을 작성할 때마다 에러 처리 로직을 작성하는 데 불편함을 느꼈다.</div>
  <br />
  <div>따라서, 중복되는 에러 핸들링 로직을 제거하기 위해 공통 로직을 분리하려고 한다.</div>
  <div>모든 mutation에 대한 에러를 전역에서 처리하려면 <a href="https://tkdodo.eu/blog/react-query-error-handling#the-global-callbacks">QueryCache에서 처리하는 것을 권장</a>한다.</div>
  <div>하지만 현재 Provider 구조상 QueryCache를 사용할 수 없다.</div>
  <div>왜 QueryCache를 사용할 수 없으며, 어떻게 공통 로직으로 분리하였는지 알아보자.</div>
</div>

## ❌ Query Cache를 사용할 수 없는 이유

처음에는 공통되는 에러 핸들링 로직을 QueryCache에서 처리하려고 했다.

먼저 에러 처리 로직을 보면 네트워크 에러일 때는 Toast, API 에러일 때는 Modal을 사용한다.

```tsx
onError: (error) => {
  if (error instanceof NetworkError) {
    show(error.message);
    return;
  }
  showModal(AlertModal, { title: '에러', message: error.message });
},
```

따라서 Context API로 내려주고 있는 Toast와 Modal을 QueryCache에서 사용하려면, <mark class="mark">ToastProvider와 ModalProvider가 QueryClientProvider보다 부모에 위치해야 한다.</mark>

QueryCache는 QueryClient를 선언할 때만 같이 선언할 수 있기 때문이다.

하지만 방 설정 모달에서 `invalidateQueries` 함수를 통해 QueryClient를 사용하기 때문에, <mark class="mark">QueryClientProvider도 Modal보다 부모에 위치해야 하는 모순이 발생한다.</mark>

> Modal 내에서 query 관련 로직을 다루면 에러 처리 로직을 분리할 수 없는건가?

## 🔍 해결 방법

머릿속을 정리하기 위해 현재 Provider 구조를 그려보니 해결책을 찾을 수 있었다.

현재 구조 : Toast ➡️ Modal ➡️ QueryClient ➡️ Modal

하지만 이 중에서도 `QueryClient ➡️ Modal 관계` 는 순서가 바뀌면 안된다. <mark class="mark">하나의 QueryClient를 공유해야 의도한대로 동작할 것이기 때문이다.</mark>

<img width="650" src="https://github.com/user-attachments/assets/71998a33-c130-4929-9ef4-7bcfc7c4e3db" />

> 그렇다면 ModalProvider 하위에서 queryClient의 설정을 바꿀 수 있는 방법은 없을까?

QueryCache를 사용할 순 없었지만 다행히도 `queryClient.setDefaultOptions` 를 이용해 기본 동작을 덮어쓰는 기능이 있었다.

이를 통해 <mark class="mark">ModalProvider 하위에서 queryClient의 기본 동작에 에러 핸들링 로직을 추가하여 공통 에러 처리 로직을 분리</mark>할 수 있었다.

---

추가로 테스트 코드에서도 에러 UI 테스트가 존재하기 때문에 해당 컴포넌트를 사용하였다.

그런데 에러 폴백 테스트 코드가 계속 통과를 안해서 힘들었는데, 알고보니 <mark class="mark">테스트 코드에서 설정해둔</mark> `retry:false` <mark class="mark">옵션이 덮어씌워져서 문제가 생긴 것</mark>이였다. 이를 테스트 환경에서만 해당 옵션이 동작하도록 분기처리하였다.

```tsx
// QueryClient는 모든 Provider에 공유되면서 공통 에러 핸들링 로직에 Toast와 Modal을 넣기 위해 setDefaultOptions 사용
// 테스트 환경에서 retry 값이 있을 경우 에러 폴백 테스트가 돌지 않아 분기 처리
const QueryClientDefaultOptionProvider = ({ children }: PropsWithChildren) => {
  const queryClient = useQueryClient();
  const { show } = useToast();
  const { show: showModal } = useModal();

  queryClient.setDefaultOptions({
    queries: {
      retry: process.env.NODE_ENV === "test" ? false : 3,
      throwOnError: true,
    },
    mutations: {
      onError: (error) => {
        if (error instanceof NetworkError) {
          show(error.message);
          return;
        }
        showModal(AlertModal, { title: "에러", message: error.message });
      },
      throwOnError: (err) => {
        const error = err as CustomError;
        return isServerError(error.status);
      },
      networkMode: "always",
    },
  });

  return <>{children}</>;
};
```

## ✅ 결론 (해결)

<ol class="blue-box">
<div>1. 공통으로 묶어서 에러 처리를 할 수 있는데, QueryClient의 QueryCache에서 해야한다.</div>
<div>2. 공통 에러 처리에 Toast와 Modal을 사용한다.</div>
<div>3. 사용하려면 Toast와 Modal이 QueryClient 보다 부모에 위치해야 한다.</div>
<div>4. 하지만 Modal에서도 QueryClient를 사용하기 때문에 QueryClient가 Modal보다 부모에 있어야 한다.</div>
<div>5. 4번이 성립하기 위해선 무조건 ModalProvider 부모에 QueryClient가 있어야 한다.</div>
<div>6. 따라서 ToastProvider ➡️ QueryClient ➡️ ModalProvider 상태로 두고, QueryClient 공통 에러 처리 로직을 ModalProvider 하위에서 처리할 수 있는 방법을 찾았다.</div>
<div>7. 6번 방식을 가능하게 하는 함수 ➡️ <span class="high">queryClient.setDefaultOptions</span></div>
</ol>

_뿌듯해지는 코멘트 ㅎㅎㅎ_

<img width="500" alt="image" src="https://github.com/user-attachments/assets/327463af-e7a9-4674-904a-ec3b8dc9ffcf">

## 📘 reference

- [setDefaultOptions - Tanstack](https://tanstack.com/query/v5/docs/reference/QueryClient/#queryclientsetdefaultoptions)
- [QueryCache - Tanstack](https://tanstack.com/query/latest/docs/reference/QueryCache)
- [공통 로직 개선 PR - 땅콩](https://github.com/woowacourse-teams/2024-ddangkong/pull/365)
