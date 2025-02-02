---
title: "Suspense와 ErrorBoundary로 관심사를 분리하여 컴포넌트 복잡도 개선하기"
layout: single
excerpt: Suspense와 ErrorBoundary를 활용해 관심사를 분리하여 컴포넌트 복잡도를 낮추고, Suspense 동작 원리를 직접 만들면서 이해한다.

categories:
  - trouble-shooting
tags: [Suspense, ErrorBoundary, 관심사 분리]
toc_label: Suspense와 ErrorBoundary로 관심사 분리
toc: true
toc_sticky: true
---

> [테크니컬 라이팅 강의](https://github.com/rbgksqkr/woowa-writing/blob/tech-writing/tech-writing.md)에서 작성했던 내용을 옮긴 내용입니다.

## 💭 글을 시작하며

> 일주일 전에 만든 내 컴포넌트를 이해할 수 없다...

서비스를 개발하는 프론트엔드 개발자라면 필수적으로 비동기 처리를 해야 한다. 비동기 작업을 어떻게 처리하느냐에 따라 서비스의 완성도를 결정짓기도 한다. useState, useEffect로 로딩 상태와 에러 상태를 처리하다보면 자연스럽게 컴포넌트가 복잡해진다. 컴포넌트의 복잡도를 낮출 수 없을까?

그래서 이 글의 예상 독자는 다음과 같다.

> 비동기 처리할 때 로딩 처리나 에러 처리를 고려해본 개발자
>
> React를 사용해본 개발자
>
> 컴포넌트 내에 비즈니스 로직을 커스텀훅으로 분리해본 개발자
>
> 비동기 처리를 커스텀 훅으로 분리했을 때 불편함을 느낀 개발자

사용자가 서비스를 이용할 때는 개발자가 의도한 시점에 의도한 동작만을 하지 않기 때문에, 다양한 케이스에 대해 고려해야 한다. 요구사항이 중간에 변경되어 다른 UI를 그려야할 수도 있다.

비동기 에러를 처리하는 방법에 대해 소개하고, 선언형으로 처리하면서 상태에 대한 관심사를 분리한 경험을 소개하려고 한다.`

## 📘 명령형으로 처리하기

에러를 명령형으로 처리하는 방법에 대해 먼저 알아보자.

자바스크립트에서 비동기 호출에 대한 에러를 다룰 때 `try-catch` 문을 사용한다.

```tsx
const getUser = () => {
  try {
    const response = await fetch(`URL`);
    const data = await response.json();

    return data;
  } catch (error) {
    console.error(error);
  }
};
```

위의 형태를 리액트 함수 컴포넌트에서 사용하기 어렵다.

함수 컴포넌트에서 비동기 에러를 핸들링하려면 useState와 useEffect를 활용해야 한다.

따라서 <mark class="mark">API 호출부에선 사용처로 에러 처리를 위임</mark>하고, 나아가 <mark class="mark">UI로부터 로직을 커스텀 훅으로 분리</mark>한다면 아래와 같이 구현할 수 있다.

```tsx
const getUser = async (id: number) => {
  const response = await fetch(`URL`);
  const data = await response.json();
  return data;
};

const useUserInfo = (id: number) => {
  const [data, setData] = useState<UserInfo>();
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState("");

  useEffect(() => {
    const fetchUser = async () => {
      try {
        setIsLoading(true);
        const user = await getUser(id);

        if (!ignore) {
          setData(user);
        }

        setIsLoading(false);
        setError("");
      } catch (err) {
        const error = err as Error;
        setError(error.message);
      }
    };

    let ignore = false;
    fetchUser();
    return () => {
      ignore = true;
    };
  }, [id]);

  return { data, isLoading, error };
};
```

커스텀 훅으로 데이터 페칭 로직을 추출함으로써 <mark class="mark">컴포넌트에서의 데이터의 흐름이 명확</mark>해졌다.

```tsx
export const UserInfo = ({ id }: { id: number }) => {
  const { data, isLoading, error } = useUserInfo(id);

  if (isLoading) return <LoadingFallback />;

  if (error) return <ErrorFallback error={error} />;

  return (
    <div>
      <h1>name: {data?.name}</h1>
      <h2>Email: {data?.email}</h2>
    </div>
  );
};
```

🔍 **중간 정리**

비즈니스 로직이 UI 로직과 분리되면서 컴포넌트가 깔끔해졌지만 `몇 가지 문제점이 존재` 한다.

1. 커스텀 훅에서 반환하는 로딩 상태와 에러 상태에 따라 매번 컴포넌트 내에서 분기처리가 필요하다.
2. 비동기 호출이 여러 개일 경우 이에 대한 처리가 복잡해지고, 코드를 유지보수하기 어려워진다.

> 비슷한 코드가 수많은 컴포넌트 내에 위치하는 게 적절한가?

## 📘 선언형으로 처리하기

커스텀 훅으로 데이터 페칭 로직을 분리했지만, 컴포넌트에 상태에 따른 분기처리가 추가되면서 로직이 <mark class="mark">명령형</mark>으로 이뤄져 있다.

> isLoading이 true일 때 LoadingFallback을 반환하고, error가 있을 때 ErrorFallback을 반환하고, 성공 케이스일 때 원하는 데이터를 반환한다.

해당 로직이 현재는 문제가 되지 않는다. 문제라고 느끼지 못할 수도 있다.

하지만 비동기 호출이 여러개일 때 각각의 로딩 상태와 에러 상태에 따라 다르게 처리하거나, 문제가 발생했을 때 에러를 추적하기 어렵다.

또한 컴포넌트 내에 상태에 따라 분기처리하는 로직이 많아지면 유지보수 관점에서 좋지 않다. 따라서, <mark class="mark">선언형으로 각각의 상태를 관심사별로 분리</mark>해보자.

### ✅ Suspense

React 18 부터 Suspense는 React.lazy 뿐만 아니라, 모든 비동기 작업을 처리할 수 있게 되었다. (코드, 데이터, 이미지 로드 등)

따라서, Suspense를 활용하면 명령형으로 처리하고 있던 <mark class="mark">비동기 로딩 상태를 선언형으로 처리</mark>할 수 있다.

_비동기 호출이 발생하는 컴포넌트를 Suspense로 감싸면, 로딩 상태일 때 fallback UI를 보여주고, 비동기 호출이 완료되면 자식 컴포넌트를 렌더링한다._

공식문서의 말을 빌리면, 개념적으로 catch 문과 유사하지만 오류를 잡는 대신 일시 중지된 컴포넌트를 잡는다.

> Conceptually, you can think of `Suspense` as being similar to a `catch` block. However, instead of catching errors, it catches components "suspending".

🔍 **Suspense 적용 예시**

비동기 로딩 상태를 Suspense로 관리한 예시를 살펴보자.

Suspense를 적용한 간단한 예시를 보여주기 위해, tanstack-query의 `useSuspenseQuery` 를 사용하였다. 앞에서 나온 예시대로 <mark class="mark">useEffect를 사용하면 Suspense를 적용할 수 없다.</mark> 이에 대한 자세한 내용은 바로 다음 문단에서 설명할 것이다.

```tsx
const App = () => {
  return (
    <Suspense fallback={<LoadingFallback />}>
      <UserInfo />
    </Suspense>
  );
};

const UserInfo = () => {
  const { data: user } = useSuspenseQuery({
    queryKey: ["user"],
    queryFn: getUser,
  });

  return <div>{user.name}</div>;
};
```

> 이로써 비동기 호출의 로딩 처리를 Suspense에 위임하여 관심사를 분리할 수 있었다.

### 🚨 Suspense는 어떻게 비동기 호출을 감지하여 fallback UI를 렌더링하는가?

Suspense 적용 예시에서 useEffect를 사용하면 Suspense를 적용할 수 없다고 했다.

그럼 Suspense는 꼭 라이브러리를 써야하는 건가? 어떻게 비동기 호출을 감지하는 걸까?

> **생각의 흐름대로 핵심을 파악해보자.**
>
> 1.  비동기 호출을 하는 자식 컴포넌트가 부모 컴포넌트에게 `무언가` 를 줘야 부모 컴포넌트인 Suspense가 이를 감지할 것이다.
> 2.  `무언가` 는 로딩 상태와 완료 상태를 모두 갖고 있어야 한다. 그래야 로딩 상태일 때 fallback UI를 렌더링하고, 완료 상태일 때 자식 컴포넌트를 렌더링할 것이다.
> 3.  ➡️ 리액트는 자바스크립트의 비동기 작업을 처리하는 객체인 `Promise` <mark class="mark">를 활용하여 Suspense에서 비동기 호출을 감지하도록 구현</mark>하였다.

<mark class="mark">Promise 객체는 pending, fulfilled, rejected 3가지 상태를 갖고 있기 때문에</mark> 로딩 상태와 완료 상태에 대한 분기처리가 모두 가능하다.

그럼 라이브러리 없이 Suspense에 감지되도록 비동기 호출을 한다면 어떻게 처리해야 할까?

> 핵심은 **Promise를 캐치**하고, **로딩 상태를 관리하는 컴포넌트**라는 것
>
> 1. 비동기 호출이 발생하는 즉시 Promise를 throw하여, Suspense가 Promise의 pending 상태를 감지하도록 한다.
> 2. Suspense 내부에서 로딩 상태를 관리하고, 로딩 상태면 fallback, 아니면 children 반환한다.
> 3. Promise가 resolve되면 children 반환한다.

이를 간단하게 구현해보면 다음과 같다.

```tsx
const cache: Record<number, { data?: User; promise?: Promise<void> }> = {};

const App = () => {
  return (
    <Suspense fallback={<LoadingFallback />}>
      <UserInfo />
    </Suspense>
  );
};

const UserInfo = ({ user }: { user: User }) => {
  const user = useUserInfo(1);

  return <h1>name: {user.name}</h1>;
};

const useUserInfo = (id: number): UserInfo => {
  if (!cache[id]) {
    const promise = getUser(id).then((data) => {
      console.log("resolve promise");
      cache[id] = { data };
    });

    console.log("throw promise");
    cache[id] = { promise };
    throw promise;
  }

  if (cache[id].promise) {
    throw cache[id].promise;
  }

  console.log("return cache data", cache[id].data!);
  return cache[id].data!;
};
```

<img width="600" src="https://github.com/user-attachments/assets/93078601-4dfb-40ea-b9f5-0de9e50ca819">

최종적으로 Suspense가 fallback UI와 자식 컴포넌트를 렌더링하는 흐름은 다음과 같다.

> 비동기 호출 발생 → Promise throw → fallback UI → Promise resolve → 컴포넌트 리렌더링 트리거 → children 렌더링

따라서 <mark class="mark">Promise가 pending 상태일 때 fallback UI, fulfilled 상태일 때 children을 반환하게 되는 것</mark>이다.

### ✅ ErrorBoundary

로딩 처리를 Suspense에 위임하였다면 이번에는 <mark class="mark">에러 처리를 ErrorBoundary에게 위임</mark>해보자.

ErrorBoundary는 `하위 컴포넌트 트리의 어디에서든 깨진 컴포넌트 트리 대신 폴백 UI를 보여주는 컴포넌트` 다.

렌더링 도중 생명주기 메서드 및 그 아래에 있는 전체 트리에서 에러를 잡아낸다.

---

기본적으로 애플리케이션이 <mark class="mark">렌더링 도중 에러를 발생시키면 React는 화면에서 해당 UI를 제거</mark>한다.

이를 방지하기 위해 UI의 일부를 ErrorBoundary로 감싸면, <mark class="mark">에러가 발생한 부분 대신 fallback UI를 표시</mark>할 수 있다.

아래는 React 공식문서에서 기본적으로 제공해주는 ErrorBoundary 클래스 컴포넌트다.

이를 커스텀하려면 추가적인 공부가 필요하고, 함수형 컴포넌트로 사용하고 싶다면 `react-error-boundary` 라이브러리를 설치하여 구현할 수 있다.

기본적인 fallback UI만 보여준다고 한다면 render함수에서 hasError가 true일 때 반환하는 JSX에 ErrorFallback 컴포넌트를 추가하면 된다.

```tsx
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    // 다음 렌더링에서 폴백 UI가 보이도록 상태를 업데이트 합니다.
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    // 에러 리포팅 서비스에 에러를 기록할 수도 있습니다.
    logErrorToMyService(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      // 폴백 UI를 커스텀하여 렌더링할 수 있습니다.
      return <h1>에러 폴백 UI</h1>;
    }

    return this.props.children;
  }
}
```

ErrorBoundary를 구현하고 에러를 잡을 컴포넌트를 감싼다.

아래와 같이 감싸주기만 하면 TodoInfo에서 에러가 발생했을 때 ErrorBoundary의 fallback UI를 보여줄 수 있다.

```tsx
const TestApp = () => {
  return (
    <ErrorBoundary>
      <TodoInfo />
    </ErrorBoundary>
  );
};
```

🔍 **Suspense와 ErrorBoundary 적용 예시**

이렇게 로딩 상태와 에러 상태를 Suspense와 ErrorBoundary에 위임함으로써 관심사를 분리할 수 있다.

TodoInfo 컴포넌트에서 분기처리되던 상태를 외부로 위임하여 컴포넌트는 성공한 케이스의 로직만 가지고 있게 되었다.

이러한 구조는 <mark class="mark">추후 요구사항이 변경되거나 다른 사람이 코드를 수정할 때 빠르게 맥락을 파악</mark>할 수 있다.

```tsx
const TestApp = () => {
  return (
    <ErrorBoundary>
      <Suspense fallback={<LoadingFallback />}>
        <TodoInfo />
      </Suspense>
    </ErrorBoundary>
  );
};
```

🚨 **ErrorBoundary가 비동기 에러를 잡지 못하는 이유**

React 공식문서에서는 아래와 같은 상황에서 ErrorBoundary 가 에러를 잡지 못한다고 설명한다.

서버 사이드 렌더링을 제외하면 모두 <mark class="mark">비동기 처리에서 에러가 발생하면 ErrorBoundary에서 에러를 포착하지 못한다는 내용</mark>이다.

비동기 에러를 못잡는 이유를 생각해보자.

<mark class="mark">비동기 작업은 콜스택이 비워진 다음 실행</mark>되는데, 비동기 로직에서 에러가 발생한다면 <mark class="mark">ErrorBoundary 경계 내에 위치하지 않게 되므로 에러를 잡지 못하게 되는 것</mark>이다.

<img width="600" alt="new1" src="https://github.com/user-attachments/assets/9f5cb0be-041b-4229-ba3e-840ebca55b61">

# 어떤 에러를 처리할 수 있을까?

앞에서 각각의 상태에 따라 선언적으로 처리하는 방법에 대해 알아보았다.

이제는 역할에 맞게 관심사를 분리하여 컴포넌트 내부를 깔끔하게 유지할 수 있게 되었다.

나아가 다양한 에러 상태를 효과적으로 다루기 위해서 어떤 에러 종류들이 존재하는지 알아보자.

## 📘 예상 가능한 에러 vs 예상 불가능한 에러

**에러가 언제 어떻게 발생할 지를 예상할 수 있는지**에 대한 관점으로 에러를 바라볼 수 있다.

특정 시점에 발생한 에러를 예측하고 대비할 수 있는지를 기준으로 예상 가능한 에러와 예상 불가능한 에러로 나눌 수 있다.

### ✅ 예상 가능한 에러

예상 가능한 에러란 **애플리케이션** **실행 전에 개발자가 미리 예상하고 대응할 수 있는 에러**다.

해당 에러는 주로 `외부 환경` 이나 `사용자 입력` 에 의해 발생한다.

try-catch 문 또는 ErrorBoundary로 예상 가능한 에러를 처리할 수 있다.

> - **사용자 입력 오류** : 잘못된 이메일 형식 또는 잘못된 필드를 제출한 경우
> - **잘못된 페이지 접근 오류** : URL로 잘못된 경로에 접근하는 경우

권한이 없거나 잘못된 접근에 대한 에러를 상황에 맞게 처리할 수 있다.

401, 403 등의 HTTP status code 내에서도 에러 코드를 정의하여 다양하게 로직을 처리할 수 있다.

### 🔍 프로젝트 적용 예시

프로젝트에서는 폼 형식으로 제출하는 영역이 적어 잘못된 페이지를 접근하는 경우에 대해 처리하였다.

`참여할 수 없는 방에 접근`하는 경우, 좌측처럼 잘못된 링크에 접속했다는 안내 문구가 뜬다. 다시 서비스를 진행할 수 있도록 추가적인 가이드를 제공할 예정이다.

`없는 페이지에 접근`하는 경우, 우측처럼 페이지 이동 시 에러가 발생했다는 안내 문구와 홈화면으로 가는 가이드를 제공한다.

|                              참여할 수 없는 방에 접근하는 경우                              |                                 없는 페이지에 접근하는 경우                                 |
| :-----------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------: |
| <img src="https://github.com/user-attachments/assets/77c30926-f348-47c4-b570-584eb13ee81a"> | <img src="https://github.com/user-attachments/assets/25aa3ad4-490b-4ed4-ac85-34337c394682"> |

### ✅ 예상 불가능한 에러

개발자가 **통제할 수 없는 외부 요인이나 예측하기 힘든 상황에서 발생하는 에러**다.

서버 API로부터 전달받는 에러 중 500번대 에러를 예측할 수 없는 에러로 분류한다.

> - **네트워크 오류 :** 네트워크가 일시적으로 중단되거나 타임아웃이 발생하는 경우
> - **런타임 타입 오류 :** 서버 장애로 API 응답이 없거나 잘못된 형식의 데이터를 반환하는 경우

같은 내용도 다른 관점에서 바라보면 예상 가능한 에러에서 예상 불가능한 에러로 나눌 수 있다.

`일시적인 네트워크 오류` 는 어느 정도 예상하여 개발 단계에서 처리할 수 있지만, <mark class="mark">언제 어떻게 발생할 지를 예측할 수 없기 때문에</mark> 해당 기준에서는 예상 불가능한 에러로 분류하였다.

예상 불가능한 에러는 <mark class="mark">ErrorBoundary 를 활용해 에러 폴백</mark>을 제공하거나 <mark class="mark">Sentry 와 같은 모니터링 시스템</mark>을 통해 대응책을 마련할 수 있을 것이다.

### 🔍 프로젝트 적용 예시

예상 불가능한 에러를 ErrorBoundary를 활용하여 처리하였다.

<mark class="mark">런타임 에러와 API 에러를 잡는 ErrorBoundary를 각각 분리</mark>하였고, Sentry로 모니터링 시스템을 구축해 에러 단계를 구분하여 Discord로 알림이 오도록 설정하였다.

tanstack-query의 `useQueryErrorResetBoundary` 를 활용하면 가장 가까운 QueryErrorResetBoundary 컴포넌트 하위에 있는 모든 쿼리 오류를 재설정한다.

현재는 일부만 fallback UI를 띄우는 상황이 없어서 별도로 설정하지 않아 기본값인 전역으로 설정되었다.

|                                            ErrorFallback UI                                             |                                           ErrorBoundary 코드                                            |
| :-----------------------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------------------: |
| <img width="350" src="https://github.com/user-attachments/assets/7eae708f-4c93-425f-a6b2-395b93896b06"> | <img width="500" src="https://github.com/user-attachments/assets/41ba7327-1f22-4dc1-8267-b102e2fc49bd"> |

## 📘 해결 가능한 에러 vs 해결 불가능한 에러

**에러 발생 후 사용자가 즉시 해결할 수 있는지**에 대한 관점으로 에러를 바라볼 수 있다.

### ✅ 해결 가능한 에러

사용자가 직접 해결하거나, 해당 에러에 대한 처리 로직이 구현되어 있어 복구할 수 있는 에러다.

에러가 발생하더라도 적절한 조치를 통해 프로그램의 정상적인 흐름으로 돌아갈 수 있다.

> - **권한 문제** : 인증 토큰이 만료되었을 때 로그인 화면 라우팅 또는 로그인 요청
> - **API 호출 실패** : 적절한 피드백을 제공하여 문제가 발생했음을 알리고 안내 메세지 출력

사용자가 서비스를 이탈하지 않도록 에러 상황을 해결할 수 있는 가이드를 제공한다.

사용자에게 액션을 가이드하지 않더라도 문제 상황을 알려줌으로써, 해결할 수 있는 상황인지를 사용자가 판단할 수 있도록 안내한다.

### 🔍 프로젝트 적용 예시

에러가 발생했을 때 <mark class="mark">사용자에게 알려야하는 에러라면 모달로 안내 메세지를 제공</mark>한다.

아래 예시는 투표 시간이 지난 후에 투표를 하여 발생한 에러를 모달로 안내하는 상황이다.

<mark class="mark">tanstack-query의 mutation에서 에러 핸들링 로직을 처리하여 에러가 전파되지 않도록 구현</mark>하였다.

|                                사용자에게 알려야하는 에러를 모달로 제공                                 |                                      에러 핸들링 코드                                       |
| :-----------------------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------: |
| <img width="650" src="https://github.com/user-attachments/assets/be969de7-7106-4120-8e14-79618dadb98f"> | <img src="https://github.com/user-attachments/assets/693dddca-8a03-4b36-abfd-8a49711daaab"> |

### ✅ 해결 불가능한 에러

말그대로 사용자가 해결할 수 없는 에러다.

서비스를 정상적으로 사용할 수 없는 상태로, 사용자에게 어떤 에러 상황인지 말해줘도 도움이 되지 않는 에러다.

> - **사용자 환경 문제** : 저사양 기기 또는 브라우저에서 동작하지 않는 코드가 포함되어 있는 경우

이렇게 에러 상태를 4가지로 나눠서 각각 어떻게 처리했는지 알아보았다. 앞에서 설명한 예시들을 통해 에러를 처리할 때 <mark class="mark">코드 레벨이 아닌 사용자 관점에서 더 생각해볼 수 있다.</mark>

## 🔥 마무리

로딩 상태와 에러 상태를 선언적 컴포넌트에 위임하여 관심사를 분리하는 과정을 살펴보았다.

단순한 컴포넌트에서는 위의 작업이 크게 의미 없을 수 있다. 서비스의 규모가 작을 경우 빠르게 구현하는 것이 더 중요할 수 있다.

하지만 단순한 분리가 아니라 서비스 규모가 커지고 비즈니스 로직이 복잡해졌을 때를 상상해본다면 관심사 분리의 필요성을 느낄 수 있을 것이다.

그렇다면 해당 글에서 전달한 Suspense와 ErrorBoundary에 대한 이해를 바탕으로 더 좋은 구조를 설계할 수 있을 것이다.

나아가 Suspense를 더 공부하고 싶다면 서버 사이드 렌더링에서 Suspense를 다루는 방법과 `Streaming SSR` 을 공부해보는 것을 권장한다.

## 📘 래퍼런스

- [효율적인 프런트엔드 에러 핸들링](https://jbee.io/articles/react/%ED%9A%A8%EC%9C%A8%EC%A0%81%EC%9D%B8%20%ED%94%84%EB%9F%B0%ED%8A%B8%EC%97%94%EB%93%9C%20%EC%97%90%EB%9F%AC%20%ED%95%B8%EB%93%A4%EB%A7%81)

---

**Suspense 공식문서**

- [리액트 공식문서 - suspense](https://ko.react.dev/reference/react/Suspense)
- [리액트 공식문서(legacy) - suspense](https://17.reactjs.org/docs/concurrent-mode-suspense.html)
- [리액트 공식문서 - react 18 suspense new feature](https://ko.react.dev/blog/2022/03/29/react-v18#new-suspense-features)
- [리액트 18 릴리즈 노트 - suspense](https://github.com/reactjs/rfcs/blob/main/text/0213-suspense-in-react-18.md)

---

**ErrorBoundary 공식문서**

- [리액트 공식문서 - error boundary로 렌더링 에러 잡기](https://ko.react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary)
- [리액트 공식문서(legacy) - ErrorBoundary](https://ko.legacy.reactjs.org/docs/error-boundaries.html)
- [Use react-error-boundary to handle errors in React - Kent C. Dodds](https://kentcdodds.com/blog/use-react-error-boundary-to-handle-errors-in-react)

---

**Suspense 내부 동작 원리 및 구현**

- [https://velog.io/@shinhw371/React-suspense-throw](https://velog.io/@shinhw371/React-suspense-throw)
- [https://maxkim-j.github.io/posts/suspense-argibraic-effect/](https://maxkim-j.github.io/posts/suspense-argibraic-effect/)
- [https://velog.io/@tap_kim/react-learn-suspense](https://velog.io/@tap_kim/react-learn-suspense)
