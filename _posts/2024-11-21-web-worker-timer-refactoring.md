---
title: "[Worker] Web Worker를 활용하여 탭 전환 시 발생하는 타이머 오차 개선하기"
layout: single
excerpt: 문제 상황을 이해하며, Web Worker의 필요성을 이해할 수 있다.

categories:
  - trouble-shooting
tags: [Web Worker, worker thread]
toc_label: Web Worker를 활용하여 탭 전환 시 발생하는 타이머 오차 개선하기
toc: true
toc_sticky: true
---

<div class="red-box">
  <div><span class="high">Web Worker</span>란 <mark class="mark">스크립트 연산을 웹 어플리케이션의 메인 스레드와 분리된 별도의 백그라운드 스레드에서 실행할 수 있는 기술</mark>입니다.</div>
  <div>따라서 <mark class="mark">Web Worker를 사용하면 브라우저 탭이 비활성화 되어도 영향을 받지 않고</mark> 멀티 스레딩처럼 동작합니다.</div>
  <div>또한 무거운 작업을 분리된 스레드에서 처리하면 메인 스레드(보통 UI 스레드)가 멈추거나 느려지지 않고 동작할 수 있습니다.</div>
</div>

반영 PR: [https://github.com/woowacourse-teams/2024-ddangkong/pull/406](https://github.com/woowacourse-teams/2024-ddangkong/pull/406)

## 📘 브라우저에서 탭 전환 시 setInterval 실행 간격에 오차가 발생하는 이유

> 브라우저가 `비활성 탭에서의 리소스 사용을 줄이기 위해` 타이머의 실행 빈도를 조절한다.

### ✅ 비활성 탭의 타임아웃

백그라운드 탭으로 인한 부하와 그로 인한 배터리 소모를 줄이기 위해, <mark class="mark">브라우저는 비활성 탭에서의 지연 시간에 최소 값을 강제</mark>한다.

Chrome의 경우 비활성 탭에 대해 `최소 1초`의 지연 시간을 강제하여 타이머 오차가 발생하게 된다.

<img width="600" src="https://github.com/user-attachments/assets/adc93a68-c14e-4875-91f8-aa30fd075e37">

### ✅ Chrome 타이머 정책

JS 타이머에 대한 Chrome 정책에서 최소 제한, 제한, 집중적인 제한을 소개하고 있다.

우리 서비스의 경우 타이머 체인이 5번 이상이며, 5분 이상의 타이머가 없으므로 `제한` 에 속한다.

따라서 브라우저는 타이머를 `초당 한 번씩 확인`하고, 제한 시간이 유사한 타이머는 함께 일괄 처리된다.

<img width="600" src="https://github.com/user-attachments/assets/2ae46cdc-f02d-4935-8997-30a8f6612a0c">

## 🚨 Worker를 사용할 때 주의할 점

Web Worker는 정말 필요할 때만 사용해야 한다.

<mark class="mark">메모리는 유한</mark>하기 때문에, 간단한 로직이라면 싱글 스레드로도 충분한데 굳이 브라우저의 백그라운드 스레드를 사용하는 건 위험하다.

멀티 스레드를 사용하게 되면 컨텍스트 스위칭이 자주 발생하면서 오버헤드 비용이 발생할 수 있고 성능 문제가 발생할 수 있다.

<div class="blue-box">
  <div>하지만 내 목표는 탭이 전환되어 비활성 탭이 되더라도 타이머를 일정하게 동작시켜 사용자 경험을 개선하는 것이므로 적절하다고 판단하였다.</div>
</div>

## 😎 적용 결과

|                                   ❌ worker 적용 전 탭 전환 시 타이머                                   |                                   ✅ worker 적용 후 탭 전환 시 타이머                                   |
| :-----------------------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------------------: |
| <img alt="image" src="https://github.com/user-attachments/assets/53b9ca0d-7a27-4618-bbc9-7058873c3b68"> | <img alt="image" src="https://github.com/user-attachments/assets/01e35a9c-7d1a-4a2c-89df-572883f80c28"> |

## 💻 Worker 타이머 예시 코드

Worker는 main thread 의 window와는 별도의 `WorkerGlobalScope` 를 갖기 때문에 worker 내에서 window 메서드나 DOM 조작이 불가능하다.

main thread와 worker thread가 서로 데이터를 주고 받기 위해선 Message System을 사용해야 한다. (`postMessage`, `onmessage`)

<div class="blue-box">
  <div>1. <mark class="mark">main thread 에서 worker.onmessage</mark> 에 worker 의 메시지를 전달받기 위한 이벤트 핸들러를 등록한다.</div>
  <div>2. <mark class="mark">worker thread 에서 작업 처리후 postMessage</mark>로 데이터를 main thread의 worker.onmessage 에게 전달한다.</div>
</div>

**TimerWorker.ts**

```ts
let intervalId: NodeJS.Timeout;

self.onmessage = function (e) {
  const { type, delay } = e.data;

  if (type === "start") {
    const startTime = Date.now();

    intervalId = setInterval(() => {
      const elapsedTime = Date.now() - startTime;

      self.postMessage({ elapsedTime });
    }, delay);
  } else if (type === "stop") {
    clearInterval(intervalId);
  }
};
```

**useTimer.ts**

```ts
const useTimer = ({
  timeLimit,
  isSelectedOption,
  isVoted,
  vote,
}: UseTimerProps) => {
  const [leftRoundTime, setLeftRoundTime] = useState(timeLimit);
  const workerRef = useRef<Worker | null>(null);

  const isVoteTimeout = leftRoundTime <= 0;
  const isAlmostFinished = leftRoundTime <= ALMOST_FINISH_SECOND;

  useEffect(() => {
    const timerWorker = new Worker(
      new URL("./timerWorker.ts", import.meta.url)
    );
    workerRef.current = timerWorker;

    timerWorker.postMessage({ type: "start", delay: POLLING_DELAY });

    timerWorker.onmessage = () => {
      setLeftRoundTime((prev) => prev - 1);
    };

    // 타이머가 끝나기 전에 투표가 완료될 경우 clean-up
    return () => {
      timerWorker.postMessage({ type: "stop" });
      timerWorker.terminate();
    };
  }, []);

  useEffect(() => {
    if (isVoteTimeout) {
      if (isSelectedOption && !isVoted) {
        vote();
      }

      workerRef.current?.postMessage({ type: "stop" });
      workerRef.current?.terminate();
    }
  }, [isVoteTimeout, isSelectedOption, isVoted, vote]);

  return { leftRoundTime, isAlmostFinished };
};

export default useTimer;
```

## 📘 reference

- [Chrome JS 타이머 과도한 실행 제한 - Chrome 88](https://developer.chrome.com/blog/timer-throttling-in-chrome-88?utm_source=chatgpt.com&hl=ko)
- [https://medium.com/@adithyaviswam/브라우저\_setInterval\_실행\_쓰로틀링\_극복](https://medium.com/@adithyaviswam/overcoming-browser-throttling-of-setinterval-executions-45387853a826)
- [Page Visibility API - MDN](https://developer.mozilla.org/ko/docs/Web/API/Page_Visibility_API#%EB%B0%B1%EA%B7%B8%EB%9D%BC%EC%9A%B4%EB%93%9C_%ED%8E%98%EC%9D%B4%EC%A7%80_%EC%84%B1%EB%8A%A5%EC%9D%84_%EC%A7%80%EC%9B%90%ED%95%98%EB%8A%94_%EC%A0%95%EC%B1%85)
- [setTimeout - MDN](https://developer.mozilla.org/ko/docs/Web/API/Window/setTimeout#%EC%A7%80%EC%97%B0_%EC%8B%9C%EA%B0%84%EC%9D%B4_%EC%A7%80%EC%A0%95%ED%95%9C_%EA%B0%92%EB%B3%B4%EB%8B%A4_%EB%8D%94_%EA%B8%B4_%EC%9D%B4%EC%9C%A0)
- [https://d2.naver.com/helloworld/2922312](https://d2.naver.com/helloworld/2922312)
- [https://samori.tistory.com/87](https://samori.tistory.com/87)
- [https://pks2974.medium.com/Web_Worker\_간단\_정리하기](https://pks2974.medium.com/web-worker-%EA%B0%84%EB%8B%A8-%EC%A0%95%EB%A6%AC%ED%95%98%EA%B8%B0-4ec90055aa4d)
