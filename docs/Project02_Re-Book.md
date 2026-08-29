# ReBook

> **대학생 중고 전공서적 거래 플랫폼**

## 🎥 Demo

프로젝트의 전체 동작은 아래 시연 영상을 통해 확인할 수 있습니다.

- **YouTube**: [Re-Book 시연 영상](youtube.com/watch?si=rC5baI9Mo6tcP4zG&v=msiGwnS4l-Q&feature=youtu.be)

## 프로젝트 개요

ReBook은 대학생과 졸업생이 전공 서적을 거래할 수 있는 웹 플랫폼입니다.

사용자는 이메일 인증 또는 카카오 소셜 로그인을 통해 회원가입 및 로그인을 진행할 수 있으며, 상품 등록, 검색, 거래자간 채팅 등의 기능을 이용할 수 있습니다.

프로젝트 진행 중 기존 모놀리식 구조를 MSA 구조로 전환도 경험했습니다.

저는 **인증 시스템과 마이페이지 기능을 중심으로 Frontend와 Backend를 함께 개발**했으며, JWT 기반 인증/인가와 Spring Security를 이용한 로그인 기능을 구현했습니다.

---

## 프로젝트 정보

| 항목 | 내용 |
|------|------|
| 기간 | 2024.08 ~ 2024.09 |
| 인원 | 5명 (풀스택 3명 / AI 1명 / 클라우드 1명) |
| 담당 | 로그인, 회원가입, 마이페이지, JWT 인증 시스템 |
| 기술 | Java, Spring Boot, Spring Security, React, MySQL, Redis, Docker Compose, Eureka |

---

# 시스템 구성

```
    React 
      │ 
      ▼ 
Spring Boot API Gateway 
      │ 
      ▼ 
Authentication Service 
      │ 
      ▼ 
  User Service 
     │  │ 
     ▼  ▼ 
  MySQL Redis
```

회원 인증은 JWT 기반으로 처리했으며, Access Token과 Refresh Token을 분리하여 인증 흐름을 구성했습니다.

Access Token은 Authorization Header를 통해 전달하고, Refresh Token은 HttpOnly Cookie로 관리했습니다.

Refresh Token은 Redis에 저장하여 재발급 요청 시 서버에 저장된 토큰과 비교하도록 구성했습니다.

프로젝트 후반에는 서비스 확장성을 고려하여 일부 기능을 분리하고 Eureka를 활용한 서비스 등록 환경을 경험했습니다.

---

# 담당 역할

### Backend

- JWT 토큰 기반 인증 시스템 구현
- Spring Security 인증/인가 구현
- 카카오 OAuth 로그인 구현
- 이메일 인증 회원가입 구현
- Access Token / Refresh Token 관리
- Redis를 활용한 Refresh Token 및 이메일 인증 코드 관리

### Frontend

- 로그인 페이지 구현
- 회원가입 페이지 구현
- 마이페이지 UI 구현
- 사용자 정보 수정 기능 구현

---

# 기술 선택

## Spring Security

로그인 기능은 단순히 사용자를 조회하는 기능이 아니라 요청마다 인증 여부를 확인해야 했습니다.

Spring Security는 Filter 기반으로 인증 흐름을 일관성 있게 관리할 수 있고, JWT와도 자연스럽게 연동할 수 있어 인증 프레임워크로 선택했습니다.

인증이 필요한 요청은 Security Filter를 거쳐 사용자 정보를 확인한 뒤 Controller로 전달하도록 구성했습니다.

---

## JWT 기반 인증

세션 기반 로그인은 서버가 사용자 상태를 유지해야 하기 때문에 서버 확장 시 세션 관리가 필요합니다.

ReBook은 REST API 기반으로 개발했기 때문에 Stateless한 인증 방식을 적용하기 위해 JWT를 사용했습니다.

인증 구조는 다음과 같이 구성했습니다.

- Access Token : API 요청 인증
- Refresh Token : Access Token 재발급

Access Token은 Authorization Header를 통해 전달하고, Refresh Token은 HttpOnly Cookie에 저장하여 클라이언트에서 직접 접근할 수 없도록 구성했습니다.

Access Token이 만료되면 Refresh Token을 이용해 새로운 Access Token을 발급하도록 구현했습니다.

---

## Redis

Redis는 이메일 인증 코드와 Refresh Token을 관리하는 용도로 사용했습니다.

이메일 인증 코드는 Redis에 저장하면서 3분의 TTL을 적용하여 일정 시간이 지나면 자동으로 만료되도록 구현했습니다.

Refresh Token은 사용자 식별값을 기준으로 Redis에 저장하고, Access Token 재발급 요청이 들어오면 Cookie로 전달된 Refresh Token과 Redis에 저장된 토큰을 비교하여 유효성을 검증했습니다.

새로운 Refresh Token을 발급하면 Redis에 저장된 기존 토큰도 새로운 토큰으로 갱신하도록 구현했습니다.

이를 통해 Refresh Token의 서버 측 상태를 관리하고, 기존 토큰의 재사용을 방지할 수 있도록 구성했습니다.

---

## MSA

초기에는 하나의 Spring Boot 애플리케이션에서 대부분의 기능을 처리하는 모놀리식 구조로 개발했습니다.

이후 서비스의 역할을 분리하기 위해 인증 기능과 사용자 관련 기능을 별도의 서비스로 분리하고, Eureka를 활용하여 서비스 등록 및 탐색 환경을 구성했습니다.

서비스가 분리되면서 기존과 달리 서비스 간 요청에서도 인증 정보를 전달하고 검증해야 했기 때문에 JWT 전달 흐름과 인증 Filter의 동작을 함께 고려했습니다.

---

# 핵심 구현

## 카카오 소셜 로그인

OAuth 인증이 완료되면 사용자 정보를 조회한 뒤 기존 회원 여부를 확인하도록 구현했습니다.

신규 사용자는 추가 회원가입 절차를 진행하고, 기존 사용자는 JWT를 발급하여 로그인하도록 구성했습니다.

---

## 이메일 인증 회원가입

이메일 인증 코드를 발송한 뒤 인증이 완료된 사용자만 회원가입을 진행할 수 있도록 구현했습니다.

인증 코드는 Redis에 저장하고 3분의 TTL을 적용했습니다.

사용자가 입력한 인증 코드를 Redis에 저장된 코드와 비교하고, 인증에 성공하면 해당 인증 코드를 삭제한 뒤 회원가입에 사용할 수 있는 JWT를 발급하도록 구현했습니다.

이를 통해 인증되지 않은 이메일의 회원가입을 방지했습니다.

---

## Refresh Token 기반 인증 관리

Access Token은 API 요청 시 Authorization Header를 통해 전달하고, Refresh Token은 HttpOnly Cookie로 관리했습니다.

Access Token이 만료되면 Cookie로 전달된 Refresh Token의 만료 여부와 Token Category를 확인한 뒤 Redis에 저장된 Refresh Token과 비교하여 유효성을 검증했습니다.

검증에 성공하면 새로운 Access Token과 Refresh Token을 발급하고, Redis에 저장된 Refresh Token도 새로운 토큰으로 갱신하도록 구현했습니다.

---

# 트러블 슈팅

## MSA 전환 과정에서 인증 예외 처리 문제

프로젝트 후반 모놀리식 구조를 MSA 환경으로 전환하면서 인증 기능이 별도의 Auth Service로 분리되었습니다.

기존에는 하나의 애플리케이션 내부에서 인증과 비즈니스 로직이 처리되었지만, MSA 전환 이후 다른 서비스에서 Auth Service에 인증을 요청하고 인증 결과를 전달받는 구조로 변경되었습니다.

저는 Auth Service를 담당하며 서비스 간 인증 요청을 처리할 수 있도록 인증 API를 구현하고, 기존 Spring Security 인증 정보를 활용하여 인증된 사용자의 username과 role을 반환하도록 구성했습니다.

이 과정에서 인증 실패 상황에서 발생한 Exception이 기존 방식으로 처리되면서 서비스 간 요청에 적절한 오류 응답을 전달하지 못하는 문제가 발생했습니다. 인증 요청부터 Exception 발생, HTTP 응답까지의 흐름을 확인하고 인증 관련 Exception 처리 구조를 수정하여 오류 상황에서도 적절한 응답을 반환할 수 있도록 개선했습니다.

이를 통해 MSA 환경에서는 정상적인 인증 처리뿐 아니라 서비스 간 통신에서 발생하는 인증 실패와 예외 상황까지 고려하여 인증 구조를 설계해야 한다는 점을 경험했습니다.

---

## 서비스 분리 경험

초기에는 로그인 기능을 하나의 서비스에서 처리했지만, 프로젝트를 진행하면서 인증 기능과 일반 비즈니스 로직의 책임을 분리하는 방향으로 구조를 수정했습니다.

또한 이후 확장 가능성을 고려하여 알림 기능도 별도의 서비스로 분리하는 경험을 했습니다.

비록 작은 규모의 프로젝트였지만 서비스의 역할을 분리하는 과정에서 MSA 구조가 단순히 서비스를 나누는 것이 아니라 **서비스 간 책임과 인증 흐름을 함께 고려해야 한다는 점**을 경험할 수 있었습니다.

---

# 프로젝트를 통해 배운 점

ReBook 프로젝트를 진행하면서 가장 많이 고민한 부분은 인증 구조였습니다.

JWT 자체를 구현하는 것보다 **인증 정보가 어떤 흐름으로 전달되고 검증되는지**를 이해하는 것이 훨씬 중요하다는 것을 경험했습니다.

또한 Spring Security Filter 기반 인증 구조를 직접 구현하면서 요청이 Controller에 도달하기 전 어떤 과정을 거치는지 이해할 수 있었고, MSA 환경에서는 인증뿐 아니라 서비스 간 책임을 명확하게 나누는 것이 중요하다는 점도 함께 배울 수 있었습니다.

프로젝트 이후에는 새로운 기능을 구현할 때도 인증 흐름과 서비스 간 의존성을 먼저 고려하는 습관을 가지게 되었습니다.
