---
title: "[비동기] return vs return await"
layout: single
excerpt: 비동기 함수를 그냥 return하는 것과 return await하는 것의 차이에 대해 이해한다.

categories:
  - trouble-shooting
tags: [비동기, return await, await]
toc_label: return vs return await
toc: true
toc_sticky: true
---

<div class="red-box">
  <div>코드를 분석하다보니 비동기 함수를 그냥 return하는 코드를 보게 되었습니다.</div>
  <div>나는 비동기 함수를 반환할 때도 매번 await을 걸어야된다고 생각해서 <mark class="high">return await</mark>으로 작성했습니다. </div>
  <div>이유라고 한다면, 비동기 함수임을 명시적으로 나타내기 위해서 await을 붙였고 가독성이 좋다고 느꼈던 것 같습니다.</div>
  <div>But, 가독성은 개인의 경험이나 성향에 따라 꽤나 주관적이라고 생각합니다. 그럼 어떤 차이가 있을까요?</div>
</div>

## 🔥 결론

### await 키워드로 인한 성능 이슈 해결 (Chrome 72+)

기존에는 `await` 키워드로 인해 불필요한 마이크로태스크가 생성되어 성능 문제가 있었습니다.

이로 인해 eslint에도 `no-return-await` 린트 규칙이 있었습니다. (8.46.0 deprecated)

하지만 V8 엔진에서 Fast Async 최적화를 통해 성능 문제가 완전히 해결하였습니다.

해결된 V8 엔진이 적용된 `Chrome 72 이상` 부터는 성능 문제가 발생하지 않으며, 오히려 Promise를 생성하는 것보다 <mark class="mark">async/await을 권장</mark>한다고 합니다.

해당 부분은 return await으로 인한 문제가 아니라, await 키워드로 인한 문제로 성능 이슈지만 해결되어 <mark class="mark">return await이 성능 문제로 사용하면 안되는 문제는 사라졌습니다</mark>.

### return과 return await의 차이는?

1. **try/catch 내부에서 에러 처리 가능**
   - `return await`을 쓰면 호출한 함수가 에러 스택에 남아, await한 Promise가 reject될 때 그 에러를 try-catch에서 잡을 수 있음
2. **스택 트레이스 유지 (디버깅 용이)**
   - `await`를 사용하면 비동기 호출 스택 트레이스를 완전하게 보존하여 에러가 발생한 함수를 추적하기 쉬움

<div class="blue-box">
  <div>공부를 해보니 더욱더 <mark class="mark">return await을 선호</mark>하게 되었습니다.</div>
  <div>비동기 에러를 추적하는 건 참 어렵습니다. 게다가 남이 짠 코드라면 더욱더.</div>
  <div>비동기 함수가 그냥 return으로 여러번 전달된다면 추적하기 굉장히 어려울 것입니다.</div>
  <div>try/catch 내부에서 처리해야 한다면 반드시 사용해야 하는 것이고, 그게 아니라 외부로 에러를 던지는 함수더라도 호출을 따라갈 수 있는 건 디버깅에 매우 용이하다고 생각합니다.</div>
</div>

## ⚙️ V8 엔진

V8 엔진 내부적으로 성능 이슈가 발생한 이유에 대해 간략하게 사려보겠습니다.

### 문제 상황

await 하나만 호출해도 다음과 같은 과정이 엔진에서 발생하고 있었습니다.

1. V8은 값을 감싸는 wrapper Promise 생성
2. await 키워드 이후로 이어서 실행하기 위해 내부적으로 throwaway Promise를 추가 생성하여 resume 핸들러 부착
3. async 함수 실행을 멈췄다가 resolve된 후 결과값 반환

최소 3회의 마이크로태스크 틱을 순차 실행하도록 스케줄하여, 불필요한 Promise 생성과 이벤트 루프 대기 지연 발생하였습니다.

### 해결

V8에서는 ECMAScript 사양에 `promiseResolve`를 도입하여, 이미 프로미스인 값에 대해선 wrapper를 생략하는 로직을 추가하였습니다.

또한, ECMAScript 사양 변경으로 인해 API 제약이 풀려 불필요한 내부 throwaway Promise도 제거하였습니다.

- 불필요한 wrapper Promise 제거
- 2개의 추가적인 마이크로태스크 틱 수 제거 (3 → 1)
- throwaway Promise 제거

## 🧪 실험 결과

비동기 함수를 return만 했을 때는 중간에 호출된 함수들이 에러 스택 트레이스에 잡히지 않습니다.

반면, return await을 할 경우 모든 함수를 추적할 수 있습니다.

|                                                        return                                                         |                                                     return await                                                      |
| :-------------------------------------------------------------------------------------------------------------------: | :-------------------------------------------------------------------------------------------------------------------: |
| <img width="659" alt="image" src="https://github.com/user-attachments/assets/ff32740a-241d-4a6e-b2dd-9e9950c4d44b" /> | <img width="659" alt="image" src="https://github.com/user-attachments/assets/4b3177a3-9d31-4925-8b75-598766b6bdb4" /> |

```tsx
import { useState } from "react";

async function PromiseFunc(): Promise<string> {
  await new Promise((r) => setTimeout(r, 100));
  throw new Error("PromiseFunc 에러 발생!");
}

// A. 단순 return 체인 (await 없이 두 번만 리턴)
async function level1Return(): Promise<string> {
  return PromiseFunc();
}
async function level2Return(): Promise<string> {
  return level1Return();
}

// B. return await 체인 (두 단계 모두 await)
async function level1ReturnAwait(): Promise<string> {
  return await PromiseFunc();
}
async function level2ReturnAwait(): Promise<string> {
  return await level1ReturnAwait();
}

export default function App() {
  const [stack, setStack] = useState<string>("");

  const handleDoubleReturn = async () => {
    try {
      await level2Return();
    } catch (e: any) {
      setStack(e.stack);
    }
  };

  const handleDoubleReturnAwait = async () => {
    try {
      await level2ReturnAwait();
    } catch (e: any) {
      setStack(e.stack);
    }
  };

  return (
    <div style={{ padding: 20 }}>
      <h2>“이중 리턴” vs “이중 return await” 스택 비교</h2>
      <button onClick={handleDoubleReturn} style={{ marginRight: 10 }}>
        Test 이중 return
      </button>
      <button onClick={handleDoubleReturnAwait}>Test 이중 return await</button>

      <h3>에러 스택 트레이스</h3>
      <pre
        style={{
          whiteSpace: "pre-wrap",
          background: "#f0f0f0",
          padding: 10,
          borderRadius: 4,
        }}
      >
        {stack}
      </pre>
    </div>
  );
}
```

## 📘 reference

- [v8.dev - fast-async](https://v8.dev/blog/fast-async#async-functions)
- [eslint - no-return-await (deprecated)](https://eslint.org/docs/latest/rules/no-return-await)
- [향로님 블로그 - return-await에 대한 견해](https://jojoldu.tistory.com/699)
