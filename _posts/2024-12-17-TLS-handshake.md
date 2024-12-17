---
title: "[CS] TLS handshake"
layout: single
excerpt: TLS handshake가 어떻게 이뤄지는지 이해할 수 있다.

categories:
  - computer-science
tags: [CS, 네트워크]
toc_label: TLS handshake
---

## 💭 예상 질문

- TLS Handshake란 무엇인가요?
- TLS Handshake에서 디지털 인증서는 어떤 역할을 하나요?
- TLS Handshake 과정을 간단히 설명해주세요.
- TLS Handshake가 완료된 후 데이터 전송은 어떤 방식으로 이루어지나요?

<div class="red-box">
    <div>TLS는 클라이언트와 서버 간에 안전한 통신을 위해 사용되는 보안 프로토콜입니다.</div>
    <div>보통 <mark class="high">웹 브라우저와 서버 간의 통신 데이터를 암호화</mark>하며, HTTPS, FTPS 등 보안이 필요한 통신에서만 사용됩니다.</div>
    <div>데이터를 주고 받기 전, TLS handshake를 통해 서버의 무결성을 확인하고 대칭키를 전달하는 과정입니다.</div>
    <div>이 과정에서 양측은 사용할 프로토콜 버전과 암호화 방식을 맞추고, 서버는 디지털 인증서를 통해 자신의 신원을 인증합니다.</div>
    <div>TLS handshake가 완료되면 동일한 세션키를 통해 안전하게 통신할 수 있습니다.</div>
</div>

<img src="https://user-images.githubusercontent.com/34904741/139517776-f2cac636-5ce5-4815-981d-33905283bf13.png" width="400" />

## handshake 과정

1. 클라이언트는 서버에게 client hello 메시지를 담아 서버로 보낸다. 이때 클라이언트가 지원하는 프로토콜 버전과 암호화 방식을 담아 함께 보낸다.
2. 서버는 세션 ID와 CA 공개 인증서를 server hello 메시지와 함께 담아 응답한다. 통신 이후 사용할 대칭키가 생성되기 전에, CA 인증서에는 <mark class="high">클라이언트에서 handshake 과정 속 암호화에 사용할 공개키를 담고 있다.</mark>
3. 클라이언트 측은 서버에서 보낸 CA 인증서에 대해 유효한 지 CA 목록에서 확인한다.
4. 클라이언트는 난수 바이트로 <mark class="high">대칭 키를 생성하고 서버의 공개키로 암호화</mark>하여 보낸다.
5. 만약 2번 단계에서 서버가 클라이언트 인증서를 함께 요구했다면, <mark class="high">클라이언트의 인증서</mark>와 <mark class="high">클라이언트의 개인키로 암호화된 임의의 바이트 문자열</mark>을 함께 보내준다.
6. 서버는 클라이언트의 인증서를 확인 후, <mark class="high">대칭키를 자신의 개인키로 복호화</mark> 후 대칭 마스터 키 생성에 활용한다.
7. 클라이언트는 handshake 과정이 완료되었다는 finished 메시지를 서버에 보내면서, <mark class="high">지금까지 보낸 교환 내역들을 해싱 후 그 값을 대칭키로 암호화</mark>하여 같이 담아 보내준다.
8. 서버도 동일하게 교환 내용들을 해싱한 뒤 클라이언트에서 보내준 값과 일치하는 지 확인한다. 일치하면 서버도 마찬가지로 finished 메시지를 <mark class="high">이번에 만든 대칭키로 암호화하여 보낸다.</mark>
9. 클라이언트는 해당 메시지를 대칭키로 복호화하여 서로 통신이 가능한 신뢰받은 사용자란 걸 인지하고, 앞으로 클라이언트와 서버는 해당 대칭키로 데이터를 안전하게 주고받을 수 있게 된다.

## 📘 reference

- [기술 면접 준비 - github](https://github.com/gyoogle/tech-interview-for-developer/blob/master/Computer%20Science/Network/TLS%20HandShake.md)
- [TLS handshake - 티스토리](https://sunrise-min.tistory.com/entry/TLS-Handshake%EB%8A%94-%EC%96%B4%EB%96%BB%EA%B2%8C-%EC%A7%84%ED%96%89%EB%90%98%EB%8A%94%EA%B0%80?category=1055346)
