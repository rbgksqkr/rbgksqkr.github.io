---
title: "[땅콩] 땅콩은 어떤 실시간 통신 방식을 사용할까?"
layout: single
excerpt: 땅콩 서비스를 개발하면서 실시간 데이터 통신 방식에 대해 고민하고, 기술 결정에 대한 이유를 소개한다.

categories:
  - woowacourse
tags: [우아한테크코스, 땅콩, 실시간 통신, 폴링, 롱폴링, 웹소켓, SSE]
toc_label: API 응답속도 비교
toc: true
toc_sticky: true
---

> 노션에 기록했던 내용을 다듬어 옮긴 내용입니다.

<div class="red-box">
  <div>친구들과 함께 즐기는 서비스를 만들기 위해 <mark class="mark">실시간 통신</mark>이 필요하다.</div>
  <div>같은 방에 있는 모든 사용자가 같은 화면을 볼 수 있어야 하기 때문이다.</div>
  <div>어떤 실시간 통신 방식을 사용할까?</div>
</div>

**조사한 실시간 통신 방식 종류**

- 폴링 (polling)
- 롱폴링 (long polling)
- 웹소켓 (websocket)
- SSE (Server Sent Event)

## 📘 서비스 내에 실시간 데이터 처리가 필요한 부분

<div class="blue-box">
  <div>다른 사용자가 방에 들어왔는가?</div>
  <div>게임이 시작되었는가?</div>
  <div>투표가 종료되었는가?</div>
  <div>다음 라운드로 이동했는가?</div>
  <div>최종 결과 확인 후 대기화면으로 돌아갔는가?</div>
</div>

## 📘 논의

- <mark class="mark">웹소켓과 SSE는 고려 ❌</mark>

  - 0.1초 때문에 문제가 생기는 서비스가 아니기 때문에 <mark class="mark">완벽한 실시간성을 보장하지 않아도 된다.</mark>
  - 러닝 커브가 존재하며, 사용해보지 않은 기술을 무작정 도입하는 건 리소스가 크다.

## 📘 폴링 vs 롱폴링

`폴링` : 지속적으로 서버에 요청을 보내 변경을 확인하는 방식

`롱폴링` : 요청을 대기시켰다가 이벤트 발생 시 응답을 보내는 방식

---

폴링은 <mark class="mark">단순하게 fetch 요청을 계속 보내서 데이터가 변경되었는지 확인하는 방식</mark>이다.

하나를 예시로 들면, 현재 방에 접속한 사용자가 누구인지 계속 요청을 보내 확인하는 것이다.

다른 사용자가 접속하면 추가된 유저 정보를 서버에서 넘겨줘 실시간으로 화면이 업데이트되는 것이다.

---

이와 달리, 롱폴링은 <mark class="mark">요청을 보내면 서버가 그 요청을 대기</mark>시킨다.

<mark class="mark">다른 사용자가 들어왔다는 이벤트가 서버에 들어오면 해당 요청에 대한 응답을 보내는 것</mark>이다.

요청이 끝나면 커넥션을 해제하며, 응답이 오래 걸리면 Timeout 되었다가 다시 요청을 보낸다.

폴링에 비해 네트워크 요청은 줄어들지만, 커넥션을 유지하기 때문에 서버 리소스가 크다.

|                                             폴링                                              |                                            롱폴링                                             |
| :-------------------------------------------------------------------------------------------: | :-------------------------------------------------------------------------------------------: |
| <img src="https://github.com/user-attachments/assets/262d077f-bbcc-4a17-87e3-71f687cee80d" /> | <img src="https://github.com/user-attachments/assets/ea4f6ba3-79e6-4d42-9f58-0c1dabb46cff" /> |

## 📘 폴링 서버 부하 테스트

- 서버에 너무 많은 요청이 발생할 것 같아 롱폴링을 고려하다가 폴링으로 선택
- 실제로 폴링 방식이 서버에 과부하를 주는지 백엔드 팀원과 함께 테스트 진행
- 방의 최대 인원인 `12명`의 트래픽을 생성하고 2분간 게임을 진행한 결과, 약 `1.6%` CPU 사용량 차지
- 1초를 주기로 요청했을 때 CPU 80% 점유율을 기준으로 약 `600명` 동시 접속 가능성 측정
- 문제가 생기는지만 확인하기 위한 테스트이므로, <mark class="mark">정확한 성능 테스트는 추후 서버측에서 진행할 예정</mark>

|                                    폴링 부하 테스트 화면                                    |                                        EC2 대시보드                                         |
| :-----------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------: |
| <img src="https://github.com/user-attachments/assets/59cca334-fd25-4e7e-be21-49a01b61675e"> | <img src="https://github.com/user-attachments/assets/09739092-2462-475a-bae6-27406f279ef4"> |

[폴링 부하 테스트 영상 링크](https://drive.google.com/file/d/1DFomysuCNU7qx4nngh4H8wUfBFJfnqix/view?usp=sharing)

## ✅ 결론

<div class="blue-box">
  <div><span class="high">폴링</span>으로 실시간 통신 방식을 구현한다.</div>
  <div>추후에 서버 부하가 커질 경우 개선을 고민한다.</div>
</div>

## 📘 reference

- [https://ko.javascript.info/long-polling](https://ko.javascript.info/long-polling)
- [우리는 채팅을 왜 Long Polling으로 개발했는가](https://velog.io/@bruni_23yong/%EC%9A%B0%EB%A6%AC%EB%8A%94-%EC%B1%84%ED%8C%85%EC%9D%84-%EC%99%9C-Long-Polling%EC%9C%BC%EB%A1%9C-%EA%B0%9C%EB%B0%9C%ED%96%88%EB%8A%94%EA%B0%80)
