# Parrotalk

> **WebRTC 기반 1:1 AI 실시간 음성 통화 서비스**

## 프로젝트 개요

Parrotalk은 WebRTC를 기반으로 브라우저에서 별도의 프로그램 설치 없이 1:1 음성 통화를 제공하는 서비스입니다.

통화 중 음성을 실시간으로 텍스트(STT)로 변환하고, AI 추천 서버와 연동하여 상황에 맞는 추천 문구를 제공하며, AWS Polly를 이용한 TTS 기능도 함께 제공합니다.

저는 통화 기능의 핵심이 되는 **WebRTC 연결 과정, Signaling 서버, 실시간 STT 연동, 통화 페이지 구현**을 담당했습니다.

---

## 프로젝트 정보

| 항목 | 내용 |
|------|------|
| 기간 | 2024.10 ~ 2024.12 |
| 인원 | 6명 (풀스택 2명 / AI 2명 / 클라우드 2명) |
| 담당 | 통화 서버(Backend), 통화 페이지(Frontend), WebRTC Signaling |
| 기술 | React, Node.js, Express, Socket.IO, WebRTC, AWS Transcribe, AWS Polly, MySQL, Kubernetes |

---

# 시스템 구성

> Architecture Diagram 추가 예정

```
React Client

        │

        ▼

Socket.IO Signaling Server

        │

        ▼

WebRTC Peer Connection

        │

        ▼

AWS Transcribe

        │

        ▼

FastAPI AI Server

        │

        ▼

React Client
```

WebRTC는 실제 음성 데이터를 Peer 간 직접 전송하고, Socket.IO는 Peer Connection을 생성하기 위한 Signaling을 담당하도록 역할을 분리했습니다.

또한 음성 스트림은 AWS Transcribe로 전달하여 실시간 STT를 수행하고, 변환된 텍스트는 AI 서버와 연동하여 추천 문구 생성에 활용했습니다.

---

# 담당 역할

프로젝트에서는 통화 기능을 중심으로 Frontend와 Backend를 함께 개발했습니다.

### Backend

- WebRTC Signaling 서버 구현
- Socket.IO 이벤트 설계 및 처리
- AWS Transcribe Streaming 연동
- RoomManager 구현
- 실시간 음성 스트림 관리

### Frontend

- React 기반 통화 페이지 구현
- 통화 UI 설계
- PeerConnection 생성 및 관리
- MediaStream 제어
- ICE Candidate 이벤트 처리

---

# 기술 선택

## React

통화 화면은 다양한 이벤트가 동시에 발생하는 화면입니다.

마이크 On/Off, 상대 연결 상태, PeerConnection 이벤트, Socket 이벤트, STT 결과 등 여러 상태가 동시에 변경되기 때문에 상태 관리가 중요했습니다.

React는 Hook 기반으로 이벤트 흐름을 관리하기 쉽고, 팀원들의 경험과 참고할 수 있는 WebRTC 레퍼런스가 많아 React를 선택했습니다.

---

## Socket.IO

WebRTC는 음성 데이터를 전송하는 기술이며 Peer를 연결하기 위한 Signaling 기능은 직접 구현해야 합니다.

Signaling 구현 방법으로 WebSocket과 Socket.IO를 비교했으며, 최종적으로 Socket.IO를 선택했습니다.

선택한 이유는 다음과 같습니다.

- Room 기반 이벤트 관리
- 양방향 통신
- 자동 재연결(Reconnect)
- Transport Fallback 지원

Parrotalk은 1:1 통화방 단위로 이벤트를 전달해야 했기 때문에 Room 기능을 제공하는 Socket.IO가 구조적으로 적합했습니다.

---

## WebRTC

실시간 음성 통화는 지연시간이 사용자 경험에 직접적인 영향을 미칩니다.

WebRTC는 Peer 간 직접 Media Stream을 전달하므로 별도의 미디어 서버를 거치지 않아 지연시간을 최소화할 수 있다는 장점이 있었습니다.

---

# 핵심 구현

## WebRTC Handshaking

통화 요청이 시작되면 Socket.IO를 통해 Offer와 Answer를 교환하고, 이후 ICE Candidate를 주고받아 Peer Connection을 생성하도록 구현했습니다.

초기 연결이 완료된 이후에는 음성 데이터가 WebRTC를 통해 직접 전송되며 Signaling 서버는 연결 과정에서만 사용됩니다.

---

## AWS Transcribe Streaming 연동

통화 중 전달되는 음성 스트림을 AWS Transcribe Streaming API와 연동하여 실시간 STT 기능을 구현했습니다.

브라우저에서 전달받은 음성 데이터를 서버에서 Stream 형태로 변환한 뒤 AWS Transcribe에 전달하고, 반환된 텍스트를 다시 클라이언트로 전송하도록 구성했습니다.

---

# 트러블 슈팅

## AWS Transcribe Streaming 구조 개선

초기 구현에서는 음성 청크가 생성될 때마다 새로운 Streaming Session을 생성하는 방식으로 구현했습니다.

이 구조는 구현은 단순했지만 통화 시간이 길어질수록 동일한 통화에서도 Streaming API가 반복적으로 호출되는 문제가 있었습니다.

테스트 과정에서는 AWS Transcribe 사용 비용이 약 100만 원 수준까지 증가했고, Streaming Session이 반복적으로 생성되면서 음성 입력이 중간에 끊기는 현상도 확인했습니다.

문제의 원인은 Streaming Session의 생명주기를 음성 청크 단위로 관리한 구조였습니다.

이를 해결하기 위해 Streaming Session을 **Room 단위**로 관리하도록 구조를 변경했습니다.

RoomManager에서 방별 PassThrough Stream과 AbortController를 관리하고, 통화가 시작될 때 Streaming Session을 한 번만 생성하도록 수정했습니다.

이후 클라이언트에서 전달되는 audio chunk는 기존 Stream에 지속적으로 write하고, 통화가 종료될 때 Session을 종료하도록 변경했습니다.

이 구조로 변경한 이후 불필요한 API 호출을 제거할 수 있었고, Streaming 안정성과 서버 비용을 함께 개선할 수 있었습니다.

---

## 일부 NAT 환경에서 WebRTC 연결 실패

모바일 데이터 환경에서는 정상적으로 연결되었지만 일부 Wi-Fi 환경에서는 Peer Connection이 생성되지 않는 문제가 발생했습니다.

원인을 확인한 결과 STUN Server만으로는 일부 NAT 환경에서 Public Address를 정상적으로 획득하지 못했고, ICE Candidate 생성 과정이 실패하고 있었습니다.

클라우드 담당자와 협업하여 TURN Server를 구축한 뒤 ICE Server 목록에 STUN과 TURN을 함께 등록했습니다.

STUN으로 연결이 불가능한 경우 TURN Relay를 사용하도록 구성하면서 다양한 네트워크 환경에서도 안정적으로 연결할 수 있도록 개선했습니다.

이 경험을 통해 WebRTC는 단순히 API를 사용하는 것이 아니라 네트워크 환경에 대한 이해가 함께 필요하다는 점을 배웠습니다.

---

# 프로젝트를 통해 배운 점

Parrotalk 프로젝트를 진행하며 가장 크게 배운 점은 **실시간 서비스에서는 세션의 생명주기 설계가 성능과 비용을 결정한다**는 것이었습니다.

처음에는 기능 구현 자체에 집중했지만, AWS Transcribe 구조를 개선하는 과정에서 단순히 동작하는 코드보다 **리소스를 어떻게 관리할 것인지**가 훨씬 중요하다는 것을 경험했습니다.

또한 WebRTC 연결 과정에서 STUN, TURN, ICE Candidate의 역할을 직접 구현하고 문제를 해결하면서 네트워크 환경에 따라 연결 방식이 달라질 수 있다는 점도 이해하게 되었습니다.

이후 프로젝트를 진행할 때는 기능 구현에 앞서 데이터 흐름과 세션의 생명주기를 먼저 설계하려고 노력하고 있습니다.