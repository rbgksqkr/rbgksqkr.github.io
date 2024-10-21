---
title: "[우아한테크코스] CSR과 SSR 환경에서 API 응답속도 비교"
layout: single
excerpt: 사용자의 네트워크 환경이 API 응답속도에 얼마나 영향을 주는지 CSR과 SSR을 비교하며 이해한다.

categories:
  - woowacourse
tags: [우아한테크코스, SSR, CSR, API]
toc_label: API 응답속도 비교
toc: true
toc_sticky: true
---

<div class="red-box">
    <div>CSR에서는 브라우저에서 자바스크립트 번들을 실행시켜 API 요청을 보낸다.</div>
    <div>브라우저에서 API서버로 요청을 보내기 때문에 사용자 네트워크 환경이 느릴수록 API 응답이 느리다.</div>
    <br />
    <div>SSR은 SSR서버에서 API서버로 보내니까 사용자 네트워크 환경에 영향이 없지 않을까? ➡️ <span class="high">YES</span></div>
    <div>진짜 영향이 없는지 실험해보자!</div>
</div>

### ✅ CSR

<div>크롬에서 네트워크 쓰로틀링을 걸고 테스트하였다.</div>
<div>테스트 결과, 2689ms ➡️ 923ms ➡️ 404ms ➡️ 227ms 로 <mark class="mark">네트워크 환경에 API 응답 속도가 영향받는다</mark>는 것을 확인할 수 있다.</div>
<hr />

|   3G   | 느린 4G | 빠른 4G | 제한 없음 |
| :----: | :-----: | :-----: | :-------: |
| 2689ms |  923ms  |  404ms  |   227ms   |

<img width="600" alt="image" src="https://github.com/user-attachments/assets/dbbcdd3e-4dc1-4b94-bb78-f6ad61b91ab2">

```tsx
  useEffect(() => {
    const fetchData = async () => {
      console.time();
      await loadMovies(TMDB_MOVIE_LISTS.popular, setPopularMovies);
      console.timeEnd();
    };

    fetchData();
   ...
  }, []);
```

### ✅ SSR

<div>처음 연결은 별도의 통신 작업이 추가되어 네트워크 속도에 상관없이 항상 더 오래걸린다.</div>
<div>두번째 연결만 제한 없음으로 설정하고, 나머지는 3G로 설정하였다.</div>
<div>3G 설정이 제한 없음보다 빠를 때도 있는 것을 보면 <mark class="mark">사용자 네트워크 환경에 영향을 받지 않는 것</mark>을 알 수 있다.</div>
<hr />

<img width="600" alt="image" src="https://github.com/user-attachments/assets/ef11e6ba-9c02-42c9-9a64-3ace9c86471f">

```tsx
router.get("/", async (_, res) => {
  console.time();
  const popularMovies = await fetchMoviesPopular();
  console.timeEnd();
   ...
}
```

<div class="blue-box">
    <div style="font-weight: bold;">[ 결론 ]</div>
    <div>측정 결과, <span class="high">CSR에서 API 응답 속도는 사용자의 네트워크 환경에 영향을 받고, SSR 에서는 영향 받지 않는다.</span></div>
    <div>API 서버가 바쁘지 않은 이상, SSR의 API 응답속도가 더 안정적이며 빠르다.</div>
</div>

## 📘 reference

- [CSR 코드](https://github.com/rbgksqkr/ssr-basecamp/tree/rbgksqkr/csr)
- [SSR 코드](https://github.com/rbgksqkr/react-ssr/tree/step1)
