---
tags:
  - stomp
  - webSocket
---
# 개념
[STOMP Spring docs](https://docs.spring.io/spring-framework/reference/web/websocket/stomp/overview.html)   
STOMP(Simple Text Oriented Messaging Protocol)은 scripting 언어를 위해 enterprise message brokers에 연결하기 위해서 만들어 졌다. 
>enterprise message brokers란<br/> 대규모 엔터프라이즈 시스템에서 애플리케이션 간 메시지 기반 통신을 관리하는 미들웨어

일반적으로 사용되는 메시징 패턴의 최소한의 subset을 해결하도록 설계되어 있다.   
STOMP는 TCP와 WebSocket과 같은 신뢰할 수 있는 양방향 스트리밍 네트워크 프로토콜에서 사용할 수 있습니다.   
또한 STOMP는 텍스트 기반의 프로토콜이지만 binary일 수도 있습니다.

## 기본 프레임
아래는 STOMP 프레임이 Http에서 모델링 되는 기본 프로토콜 구조이다.
```text
COMMAND
header1:value1
header2:value2

Body^@
```
클라이언트는 `SEND` 또는 `SUBSCRIBE` 명령어를 사용해서 메세지를 보내거나 구독이 가능하다.   
또한 `DESTINATION`헤더를 사용해서 누가 받을 것인지 지정 해줄수 있다.   
이를 이용해서 메시지를 다른 연결된 클라이언트로 보내거나 일부 작업을 수행할 수 있도록 요청할 수 있다.   
Spring의 STOMP 지원을 사용하면 Spring WebSocket 애플리케이션은 클라이언트에게 브로커 역할을 하게 된다. 메시지는 `@Controller` message-handling methods 또는 구독을 추적하고 구독된 사용자에게 메시지를 브로드캐스트해주는 인메모리 브로커로 라우팅된다. 또한 실제 메시지 브로드캐스트를 위해서 전용 STOMP 브로커를 설정해 줄 수 있다. 이 경우에는 Spring이 브로커에 대한 TCP 연결을 유지하고 메시지를 전달하면 연결된 WebSocket 클라이언트로 메시지를 전달한다.    
# 프레임 예제
## 클라이언트 프레임
### SUBSCRIBE
서버가 주기적으로 전달할 수 있는 재고 견적을 받기 위해서 가입하는 클라이언트
```text
SUBSCRIBE
id:sub-1
destination:/topic/price.stock.*

^@
```
### SEND
거래 요청을 보내는 클라이언트
```text
SEND
destination:/queue/trade
content-type:application/json
content-length:44

{"action":"BUY","ticker":"MMM","shares":44}^@
```
### 기타 프레임
- **CONNECT/STOMP**: 연결 요청
- **UNSUBSCRIBE**: 구독 해제
- **BEGIN**: 트랜잭션 시작
- **COMMIT**: 트랜잭션 커밋
- **ABORT**: 트랜잭션 중단
- **ACK/NACK**: 메시지 수신 확인/거부
- **DISCONNECT**: 연결 종료

## 서버 프레임
### MESSAGE
STOMP 서버는 MESSAGE 명령을 사용해서 모든 구독자에게 메시지를 브로드캐스트할 수 있다.    
구독한 클라이언트에게 주식 견적을 전달하는 예제이다.
```text
MESSAGE
message-id:nxahklf6-1
subscription:sub-1
destination:/topic/price.stock.MMM

{"ticker":"MMM","price":129.45}^@
```
### 기타 프레임
- **CONNECTED**: 연결 성공 응답
- **RECEIPT**: 클라이언트 요청에 대한 수신 확인
- **ERROR**: 에러 발생 시 에러 정보 전달
## 프레임 별 헤더 정보
[프레임 별 상세 정보](https://stomp.github.io/stomp-specification-1.2.html#Frames_and_Headers)
# 장점
- 커스텀 메시징 프로토콜 및 메시지 포맷을 직접 만들 필요 없음
	- WebSocket은 기본적으로 바이너리 또는 텍스트 메시지를 송수신하는 기능만 제공
	- STOMP를 사용하면 메시지 헤더, 목적지, 구독 등의 구조를 제공하므로 별도의 프로토콜을 정의할 필요가 없음
- 다양한 STOMP 클라이언트 지원
	- STOMP는 표준화된 메시징 프로토콜이므로 Java(Spring), JavaScript, Python, C++ 등 다양한 클라이언트에서 사용 가능
- RabbitMQ, ActiveMQ 등 메시지 브로커와 연동 가능
	- 메시지 브로커(RabbitMQ, ActiveMQ 등)를 사용하면 WebSocket 서버의 부담을 줄이고 확장성을 향상 시킬 수 있다
	- 클라이언트가 브로커를 통해 메시지를 주고받으며, 다수의 사용자에게 메시지를 브로드캐스트할 수 있음
- Spring의 @Controller와 통합 가능
	- STOMP를 사용하면 WebSocket 메시지를 특정 @Controller 메서드로 라우팅 가능.
	- destination 헤더를 활용하여 메시지를 컨트롤러에서 직접 처리할 수 있음
- Spring Security를 활용한 메시지 보안
	- STOMP 메시지는 사용자 인증 및 권한 검사를 적용할 수 있음
	- 특정 사용자에게만 메시지를 전달하거나, 특정 목적지에 대한 접근을 제어 가능