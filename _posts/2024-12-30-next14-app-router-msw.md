---
title: "[Next.js] Next 14 App Router에서 msw로 API mocking 하기"
layout: single
excerpt: Nextjs에서 msw가 동작하는 원리를 이해하며, API를 모킹할 수 있다.

categories:
  - trouble-shooting
tags: [Nextjs, App Router, msw, CSR, SSR]
toc_label: App Router에서 msw 설정
toc: true
toc_sticky: true
---

> next: 14.2.22, msw: 2.7.0 기준으로 작성되었습니다.

<div class="red-box">
  <div>Nextjs에서 msw로 API를 모킹하는 방식을 이해하자.</div>
</div>

## msw 초기 설정

msw 설치 및 ServiceWorker 등록하는 파일 생성

```bash
npm i -D msw
npx msw init ./public --save
```

## Next.js 에서 설정 시 유의할 점

Next.js의 App Router의 경우 기본적으로 `서버 컴포넌트` 사용

- 서버 컴포넌트는 **서버에서 렌더링** → 브라우저 환경 ❌

<mark class="mark">MSW는 브라우저 환경에서만 동작</mark>하며, 클라이언트 측 네트워크 요청을 가로채고 이를 처리하는 방식으로 동작

- 브라우저 환경에서 네트워크 요청을 가로채기 위해 <mark class="mark">서비스 워커를 등록</mark>해야함
- 클라이언트에서 한 번만 이루어져야함

## 문제 1: msw module not found

next 14에서 msw를 실행했을 때 msw 패키지를 못찾는 오류가 발생한다.

<img width="400" src="https://github.com/user-attachments/assets/5496e197-cf90-402a-8b6c-0ffabba5cc70" />

### ✅ 해결: **MSW 패키지의 특정 경로를 무시**하도록 설정 변경

서버 환경과 클라이언트 환경에서 모두 빌드하는데, 각 환경에 맞는 파일을 탐색하기 위해 <mark class="mark">사용하지 않는 패키지의 경로를 무시</mark>하는 방식

- 서버 측에서 빌드할 땐 `msw/browser` 무시
- 클라이언트 측에서 빌드할 땐 `msw/node` 무시

---

**next config**

next.config.mjs 파일에서 webpack 커스텀 설정을 할 수 있다.

이 webpack 함수는 **세 번 실행**되며, **서버(nodejs/edge 런타임)에서 두 번, 클라이언트에서 한 번 실행**됨

이를 통해 `isServer` 속성을 통해 클라이언트와 서버를 구별할 수 있음

```js
// next.config.mjs
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  compiler: {
    emotion: true,
  },
  webpack: (config, { isServer }) => {
    if (isServer) {
      // next server build => ignore msw/browser
      if (Array.isArray(config.resolve.alias)) {
        config.resolve.alias.push({ name: "msw/browser", alias: false });
      } else {
        config.resolve.alias["msw/browser"] = false;
      }
    } else {
      // browser => ignore msw/node
      if (Array.isArray(config.resolve.alias)) {
        config.resolve.alias.push({ name: "msw/node", alias: false });
      } else {
        config.resolve.alias["msw/node"] = false;
      }
    }
    return config;
  },
};

export default nextConfig;
```

## 문제 2: SSR fetch mocking

msw는 브라우저의 서비스 워커를 통해 브라우저에서 서버로 보내는 네트워크 요청을 가로챈다.

SSR의 경우 <mark class="mark">브라우저를 통한 네트워크 요청이 아니므로 mocking이 되지 않는다.</mark>

```tsx
export default async function Home() {
  const res = await fetch("/api/test");
  const result = await res.json();
  console.log(result);

  return (
    <div>
      <span>test</span>
    </div>
  );
}
```

### ✅ 해결: 별도의 목서버를 띄워 해결

msw middleware를 이용하여 별도의 `mock server` 를 띄우고, <mark class="mark">next 서버가 보내는 네트워크 요청 mocking</mark>

→ 서버 컴포넌트에서 호출하는 API 요청도 msw로 mocking

```tsx
pnpm i -D express @types/express @mswjs/http-middleware
```

```tsx
// http.ts
import express from "express";
import { createMiddleware } from "@mswjs/http-middleware";
import { handlers } from "./handlers";

const app = express();
const PORT = 3001;

app.use(express.json());
app.use(createMiddleware(...handlers));

app.listen(PORT, () => console.log(`Mock server is running on port: ${PORT}`));
```

```tsx
// page.tsx
const BASE_URL =
  process.env.NEXT_PUBLIC_API_MOCKING === "enable"
    ? process.env.NEXT_PUBLIC_MOCK_BASE_URL
    : process.env.NEXT_PUBLIC_API_BASE_URL;

const getData = async () => {
  const response = await fetch(`${BASE_URL}/api/test`);
  return await response.json();
};

export default async function Home() {
  const res = await getData();

  return (
    <div>
      <span>{res.id}</span>
    </div>
  );
}
```

## 추가적인 오류 해결

### 오류 1: Found a redundant worker.start() call

첫 렌더링에 worker를 생성하고, initMSW 호출 후 setState가 실행되면 worker를 2번 생성한다.

불필요한 호출을 없애기 위해 ref에 worker 생성 여부를 저장하고 이미 생성했을 경우 호출을 막는다.

```tsx
[MSW] Found a redundant "worker.start()" call.
Note that starting the worker while mocking is already enabled will have no effect.
Consider removing this "worker.start()" call.
```

### 오류 2: Uncaught Error: Failed to parse URL from /api/test

상대경로는 브라우저의 현재 URL을 기준으로 동작하므로, 서버 측에서 실행되는 `fetch` 요청은 **현재 호스트 URL**을 알 수 없음

→ http 또는 https 로 시작하는 BASE_URL을 설정하여 절대 경로로 fetch 요청

## SSR fetch mocking 성공

<img width="300" src="https://github.com/user-attachments/assets/06a922dd-147e-4826-8319-8d51b3dace43" />

## 최종 반영 코드

```tsx
// mocks/browser.ts
import { setupWorker } from "msw/browser";
import { handlers } from "./handlers";

export const worker = setupWorker(...handlers);

// mocks/server.ts
import { setupServer } from "msw/node";
import { handlers } from "./handlers";

export const server = setupServer(...handlers);

// mocks/index.ts
export const initMSW = async () => {
  const isServer = typeof window === "undefined";

  if (isServer) {
    const { server } = await import("./server");
    server.listen({ onUnhandledRequest: "bypass" });
  } else {
    const { worker } = await import("./browser");
    await worker.start({ onUnhandledRequest: "bypass" });
  }
};
```

**mocks/MSWProvider.tsx**

```tsx
"use client";

import { useEffect, useRef, useState } from "react";

const isMSWEnabled = process.env.NEXT_PUBLIC_API_MOCKING === "enable";

const MSWProvider = ({ children }: { children: React.ReactNode }) => {
  const [isMSWReady, setIsMSWReady] = useState(!isMSWEnabled);
  const isStartedRef = useRef(!isMSWEnabled);

  useEffect(() => {
    const init = async () => {
      if (isStartedRef.current) return;
      isStartedRef.current = true;

      const initMSW = await import("./index").then((res) => res.initMSW);
      await initMSW();
      setIsMSWReady(true);
    };

    if (!isMSWReady) {
      init();
    }
  }, [isMSWReady]);

  if (!isMSWReady) return null;

  return <>{children}</>;
};

export default MSWProvider;
```

**app/layout.tsx**

```tsx
import Providers from "./providers";

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="en">
      <body>
        <MSWProvider>{children}</MSWProvider>
      </body>
    </html>
  );
}
```

**next.config.mjs**

```jsx
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  compiler: {
    emotion: true,
  },
  webpack: (config, { isServer }) => {
    if (isServer) {
      // next server build => ignore msw/browser
      if (Array.isArray(config.resolve.alias)) {
        config.resolve.alias.push({ name: "msw/browser", alias: false });
      } else {
        config.resolve.alias["msw/browser"] = false;
      }
    } else {
      // browser build => ignore msw/node
      if (Array.isArray(config.resolve.alias)) {
        config.resolve.alias.push({ name: "msw/node", alias: false });
      } else {
        config.resolve.alias["msw/node"] = false;
      }
    }
    return config;
  },
};

export default nextConfig;
```

**mocks/http.ts**

```tsx
import express from "express";
import { createMiddleware } from "@mswjs/http-middleware";
import { handlers } from "./handlers";

const app = express();
const PORT = 3001;

app.use(express.json());
app.use(createMiddleware(...handlers));

app.listen(PORT, () => console.log(`Mock server is running on port: ${PORT}`));
```

**app/page.tsx**

```tsx
const BASE_URL =
  process.env.NEXT_PUBLIC_API_MOCKING === "enable"
    ? process.env.NEXT_PUBLIC_MOCK_BASE_URL
    : process.env.NEXT_PUBLIC_API_BASE_URL;

const getData = async () => {
  const response = await fetch(`${BASE_URL}/api/test`);
  return await response.json();
};

export default async function Home() {
  const res = await getData();

  return (
    <div>
      <span>{res.id}</span>
    </div>
  );
}
```

## 📘 reference

- [msw module not found 해결 - github issue](https://github.com/mswjs/msw/issues/1801#issuecomment-1793911389)
- SSR fetch mocking 문제 해결
  - [https://blog.naver.com/dlaxodud2388/223433157608](https://blog.naver.com/dlaxodud2388/223433157608)
  - [https://github.com/mswjs/msw/issues/1644#issuecomment-1750722052](https://github.com/mswjs/msw/issues/1644#issuecomment-1750722052)
- [불필요한 worker 호출 해결](https://velog.io/@sarajo/Next.js-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8%EC%97%90-MSW-%EB%8F%84%EC%9E%85%ED%95%98%EA%B8%B0-API-%ED%98%B8%EC%B6%9C-%EB%B9%84%EC%9A%A9-%EC%A0%88%EA%B0%90%ED%95%98%EA%B8%B0#6-msw-%EC%BB%B4%ED%8F%AC%EB%84%8C%ED%8A%B8-%EC%83%9D%EC%84%B1)
- 참고 블로그
  - [https://www.handongryong.com/post/msw/](https://www.handongryong.com/post/msw/)
  - [https://velog.io/@ssoon-m/Next.js-app-directory의-서버-컴포넌트에서-msw-사용하기](https://velog.io/@ssoon-m/Next.js-app-directory%EC%9D%98-%EC%84%9C%EB%B2%84-%EC%BB%B4%ED%8F%AC%EB%84%8C%ED%8A%B8%EC%97%90%EC%84%9C-msw-%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0)
