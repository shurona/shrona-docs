---
tags:
  - jwt
---
# 개념
json 포맷을 이용하여서 사용자의 속성을 저장하는 Claim 기반의 Web Token이다.
RFC 7519 표준에 정의된 토큰으로, 클라이언트와 서버 간에 정보를 안전하게 전달하기 위한 방식입니다.

## 구성 요소
- Header
	- 토큰 타입 및 서명 알고리즘을 명시한다.
- Payload
	- 사용자 정보, 토큰의 생성 시간 및 만료 시간등을 적은 Claim 정보
- Signature
	- 헤더와 페이로드를 Secret Key로 서명하여 토큰의 무결성을 보장한다.

## 특징
- JWT는 자체적으로 사용자의 정보를 포함하므로 서버가 별도의 데이터베이스 조회 없이도 정보를 확인할 수 있다.
- 서명을 통해서 데이터가 변조되지 않았음을 확인할 수 있다.
- Base64Url로 인코딩되어 있기 때문에 누구나 내용을 볼 수 있습니다. 민감한 정보(예: 비밀번호, 개인식별번호)는 포함하지 않아야 합니다.
## 장점
- 동시 접속자가 많을 때 서버 측 부하를 낮춰준다.
- 클라이언트와 서버가 다른 도메인을 사용할 때
    - 이전에 클라이언트와 작업을 진행할 때 클라이언트와 서버의 도메인이 달라서 세션을 저장하는데 어려움을 겪었었음. 
    - JWT를 사용하면 Header에 데이터를 주고 받을 수 있으므로 그런 어려움에서 벗어날 수 있었음.
    - 이미 발행된 토큰의 관리가 어려움으로 인증 시간은 주의해야 한다.
- 서버가 상태를 저장히자 않으므로 쉽게 확장이 가능하다
## 단점
- 구현의 복잡도 증가
- JWT에 담는 내용만큼의 네트워크 트래픽 증가
- 먼저 생성된 JWT의 일부만 만료시킬 방법이 없음
- Secret key가 유출 시 조작이 가능하다

## 예제
#### plainText
``` text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6InNodXJvbmEiLCJpYXQiOjE1MTYyMzkwMjJ9.AHwLx7IdKbPjlql0SpT0k5DR8kNZ7W48H2_d-ZwjKDw
```
#### Header
```Json
{
  "alg": "HS256",
  "typ": "JWT"
}
```
#### Payload
```Json
{
  "sub": "1234567890",
  "name": "shurona",
  "iat": 1516239022
}
```
## Refresh Token
### 정의
- JWT에서 **Refresh Token**은 만료된 Access Token을 갱신하기 위해 발급되는 토큰이다.
### Flow
```mermaid
flowchart TD
    subgraph Client
        A[사용자 → 로그인 폼 제출]
        D[Access/Refresh Token 저장]
        E["API 요청 (Access Token 포함)"]
        I[401 응답 시 Refresh 요청]
        L[새 토큰 저장 및 원래 요청 재시도]
    end

    subgraph AuthServer
        B[자격 증명 검증]
        C[Access Token + Refresh Token 발급]
        J[Refresh Token 검증]
        K["새로운 Access(+Refresh) Token 발급"]
    end

    subgraph ResourceServer
        F{Access Token 유효?}
        G[요청 처리 및 데이터 반환]
        H[401 Unauthorized 반환]
    end

    A --> B
    B -- 인증 성공 --> C
    C --> D
    D --> E
    E --> F
    F -- 유효 --> G
    F -- 만료/무효 --> H
    H --> I
    I --> J
    J -- 검증 성공 --> K
    K --> L
    L --> E
```

### 정리
- jwt refresh
