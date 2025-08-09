---
tags:
  - kafka
  - offset
---
# 카프카 ack-mode에서 offset 처리
## Batch
### 개념
- Spring Kafka의 기본 ack# offset 처리 중 발생한 문제
### 특징
**장점**
- offset commit 횟수가 줄어들어 네트워크 오버헤드가 감소한다.
- 개발자가 커밋 타이밍을 신경 쓸 필요 없음

**단점**
- 배치 처리 중 애플리케이션이 중단되면
	- 배치의 일부 메시지만 처리되고 offset은 commit되지 않습니다
	- 재시작 시 이미 처리된 메시지들도 다시 처리됩니다
	- 이는 정상적인 상황으로, at-least-once 보장을 위한 동작입니다
## Record
### 개념
- 각 레코드(메시지)가 처리될 때마다 개별적으로 offset을 커밋하는 모드
- 배치 내의 각 레코드 처리가 완료되면 즉시 해당 레코드까지의 offset을 커밋
- 리스너 컨테이너가 각 레코드 처리 완료를 감지하면 자동으로 커밋하므로 별도의 `acknowledge()` 호출 불필요
### 특징
**장점**
- 메시지 손실 위험을 최소화 (각 레코드별로 커밋)
- 애플리케이션 중단 시 재처리해야 할 메시지 수가 적음
- 정확한 처리 지점까지만 커밋되어 중복 처리 최소화

**단점**
- offset commit 횟수가 많아져 네트워크 오버헤드 증가
- Kafka 브로커에 부하 증가 (빈번한 커밋 요청)
- 처리량(throughput) 감소 가능성

**사용 시나리오**
- 메시지 처리 실패 시 재처리 비용이 높은 경우
- 정확한 offset 관리가 중요한 비즈니스 로직
- 높은 안정성이 요구되는 시스템

```Java
@KafkaListener(topics = "critical-orders")
public void processCriticalOrder(OrderEvent order) {
    // 각 레코드 처리 완료 시 자동으로 offset 커밋
    validateOrder(order);
    saveToDatabase(order);
    sendNotification(order);
    // 처리 완료 → 자동 커밋
}
```
 
## manual
### 개념
- acknowledge()가 여러 번 호출되어도, 실제 커밋은 다음 poll()에서 일괄로 처리된다.
- `acknowledge()` 호출 즉시가 아니라, 다음 poll() 시점에 batch로 커밋
### MANUAL 모드에서의 커밋
- `acknowledge()`가 호출되면 오프셋이 큐에 적재된다.
- 리스너 스레드가 배치 처리를 끝내고 다음 poll()을 호출하기 직전에, 큐에 쌓인 모든 오프셋을 한꺼번에 `commitSync()`(또는 `commitAsync()`)로 브로커에 전송한다.
- 현재 배치가 완전히 끝난 뒤에야 실제 커밋이 일어나며, 그 시점이 `다음 poll()`이다.
### 특징
```Java
// 1. acknowledge() 호출
acknowledgment.acknowledge(); // 큐에 저장됨

// 2. Spring Kafka 내부에서 처리
// - 리스너 컨테이너가 ack 요청을 내부 큐에 적재
// - 현재 배치의 모든 레코드가 처리될 때까지 대기

// 3. 다음 poll() 시점에 일괄 커밋
// - 큐에 쌓인 모든 ack들을 commitSync() 또는 commitAsync()로 처리
```
## manual_immediate
### 개념
- 애플리케이션 코드에서 `Acknowledgment.acknowledge()` 메서드가 호출되는 즉시 오프셋을 커밋하는 모드
- 개발자가 메시지 처리의 성공/실패를 직접 제어할 수 있게 해주는 수동 커밋 방식
### 특징
- `acknowledge()` 메서드가 호출되면 바로 오프셋이 커밋된다.
- 다음 poll()까지 기다리지 않고 현재 처리한 메시지의 오프셋까지 즉시 커밋된다.
- 리스터 쓰레드에서 커밋을 해야 한다.
	- 커밋 작업은 반드시 Consumer 스레드(리스너 스레드)에서만 실행되어야 한다.
```Java
@KafkaListener(topics = "order-topic")
public void processOrder(OrderEvent order, Acknowledgment ack) {
    // Case 1: 동기 처리 (리스너 스레드)
    if (order.isSimple()) {
        processSimpleOrder(order);
        ack.acknowledge(); // ✅ 즉시 커밋
        return;
    }
    
    // Case 2: 비동기 처리 (다른 스레드)
    orderProcessingService.processAsync(order)
        .thenRun(() -> {
            ack.acknowledge(); // ❌ 큐에 저장, 나중에 커밋
        });
}
```
# offset 처리 중 발생한 문제
## MANUAL 모드에서 acknowledge() 호출하지 않아도 컨슈머가 계속 진행하는 현상
### 현상 설명
MANUAL 모드에서 `acknowledge()`를 호출하지 않아도 컨슈머가 계속해서 다음 메시지들을 처리하는 상황이 발생합니다.
### 원인 분석
Spring Kafka에서는 **메시지 읽기(polling)**와 **offset commit**이 완전히 별개의 과정으로 동작하기 때문입니다.
### 메시지 처리 위치 vs Commit된 Offset의 차이
**1. 컨슈머의 현재 처리 위치 (Current Position)**
- Kafka 컨슈머가 현재 읽어서 처리하고 있는 메시지의 offset
- `acknowledge()`를 호출하지 않아도 계속 증가
- 컨슈머는 poll()을 통해 계속 다음 배치의 메시지들을 가져옴

**2. 브로커에 Commit된 Offset (Committed Offset)**
- 실제로 Kafka 브로커에 저장된 "처리 완료" 지점
- `acknowledge()` 호출 시에만 업데이트됨
- 서버 재시작 시 이 지점부터 다시 시작
### 동작 예시
```
토픽의 메시지: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

시나리오:
1. 메시지 1, 2, 3 처리 → acknowledge() 호출 → offset 3 커밋
2. 메시지 4, 5, 6 처리 → acknowledge() 호출 안함
3. 메시지 7, 8, 9 처리 → acknowledge() 호출 안함

결과:
- 컨슈머 현재 위치: offset 9까지 처리 완료
- 브로커 저장된 offset: 3 (마지막 커밋 지점)
- 서버 재시작 시: offset 4부터 다시 시작 (4,5,6,7,8,9 중복 처리)
```
