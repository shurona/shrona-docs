---
tags:
  - stomp
  - webSocket
---
# 개념
[STOMP Spring docs](https://docs.spring.io/spring-framework/reference/web/websocket/stomp/overview.html)   
STOMP(Simple Text Oriented Messaging Protocol)은 scripting 언어를 위해 enterprise message brokers에 연결하기 위해서 만들어 졌다. 
>enterprise message brokers란<br/> 대규모 엔터프라이즈 시스템에서 애플리케이션 간 메시지 기반 통신을 관리하는 미들웨어

일반적으로 사용되는 메시징 페턴의 최소한의 subset을 해결하도록 설계되어 있다.   
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
Spring의 STOMP 지원을 사용하면 Spring WebSocket 어플리케이션은 client에게 브로커 역할을 하게 된다. 메시지는 `@Controller` message-handling methods 또는 구독을 추적하고 구독된 사용자에게 메시지를 broadcasts해주는 인 메모리 브로커로 라우팅 된다. 또한 실제 메시지 boadcast를 위해서 전용 STOMP broker를 설정해 줄 수 있다. 이 경우에는 Spring은 브로커에 대한 TCP 연결을 유지하고 메시지를 전달하면 연결된 WebSocket 클라이언트로 메시지를 전달한다.    
# 프레임 예제
## SUBSCRIBE
서버가 주기적으로 전달할 수 있는 재고 견적을 받기 위해서 가입하는 클라이언트
```text
SUBSCRIBE
id:sub-1
destination:/topic/price.stock.*

^@
```
## SEND
거래 요청을 보내는 클라이언트
```text
SEND
destination:/queue/trade
content-type:application/json
content-length:44

{"action":"BUY","ticker":"MMM","shares",44}^@
```
## MESSAGE
Stomp 서버는 메세지 명령을 사용해서 모든 구독자에게 메시지를 broadcast할 수 있다.    
구독한 클라이언트에서 주식 견적을 전달하는 예제이다.
```text
MESSAGE
message-id:nxahklf6-1
subscription:sub-1
destination:/topic/price.stock.MMM

{"ticker":"MMM","price":129.45}^@
```
