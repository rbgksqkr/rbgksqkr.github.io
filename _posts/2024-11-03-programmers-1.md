---
title: "[알고리즘] 없는 숫자 더하기"
layout: single
excerpt: 알고리즘 문제 풀이 후 풀이 기법만 간단히 기록하기

categories:
  - 구현
tags: [알고리즘, JS, 구현, reduce]
toc_label: 알고리즘
toc: true
toc_sticky: true
---

- 0-9 정수 배열 numbers 입력
- numbers에 없는 0부터 9까지의 숫자를 모두 더한 수를 return
- answer = 0-9까지의 합(45) - `numbers의 합(reduce)`

### 제출 코드

```jsx
// 제출 코드
function solution(numbers) {
  var answer = 0;

  for (let i = 0; i <= 9; i++) {
    if (!numbers.includes(i)) {
      answer += i;
    }
  }

  return answer;
}
```

### 개선 코드

```jsx
function solution(numbers) {
  return 45 - numbers.reduce((acc, cur) => acc + cur, 0);
}
```

## 📘 reference

- [없는 숫자 더하기](https://school.programmers.co.kr/learn/courses/30/lessons/86051)
