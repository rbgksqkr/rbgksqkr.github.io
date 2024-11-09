---
title: "[알고리즘] 피로도"
layout: single
excerpt: 알고리즘 문제 풀이 후 풀이 기법만 간단히 기록하기

categories:
  - 백트래킹
tags: [알고리즘, JS, 구현, 백트래킹]
toc_label: 알고리즘
toc: true
toc_sticky: true
---

- dfs(현재 피로도, 현재까지 돈 던전 수)
- 방문 X & 현재 피로도가 최소 필요 피로도보다 크거나 같다 → 방문 + 피로도 감소 + 카운팅
- 카운팅 값을 max 값 구하기

### 제출 코드

```jsx
function solution(k, dungeons) {
  let n = dungeons.length;
  let m = dungeons[0].length;

  let data = [];
  let visited = Array(dungeons.length).fill(false);
  const curDungeons = pickDungeons(data, dungeons, visited, k);

  return result;
}

var result = 0;

function solve(data, k) {
  let answer = 0;
  for (let dungeon of data) {
    [required, minus] = dungeon;
    if (k >= required) {
      k -= minus;
      answer += 1;
    }
  }

  result = Math.max(result, answer);
}

function pickDungeons(data, dungeons, visited, k) {
  if (data.length == dungeons.length) {
    solve(data, k);
    return;
  }

  for (let i = 0; i < dungeons.length; i++) {
    if (!visited[i]) {
      visited[i] = true;
      data.push(dungeons[i]);
      pickDungeons(data, dungeons, visited, k);
      visited[i] = false;
      data.pop();
    }
  }
}
```

### 개선 코드

```jsx
function solution(k, dungeons) {
  var answer = -1;

  let visited = Array(dungeons.lenth).fill(0);

  function dfs(k, count) {
    // dfs(현재 피로도, 현재까지 돈 던전 수)
    answer = Math.max(answer, count);

    for (let i = 0; i < dungeons.length; i++) {
      if (!visited[i] && k >= dungeons[i][0]) {
        visited[i] = 1;
        dfs(k - dungeons[i][1], count + 1);
        visited[i] = 0;
      }
    }
  }

  dfs(k, 0);

  return answer;
}
```

## 📘 reference

- [피로도](https://school.programmers.co.kr/learn/courses/30/lessons/87946)
