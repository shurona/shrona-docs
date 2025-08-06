---
tags:
  - kafka
  - offset
---
# 카프카 ack-mode에서 offset 처리
## Batch
### 개념
- Spring Kafka의 기본값
- `poll()` 메서드로 호출된 레코드 배치가 모두 처리된 이후 커밋
- 자동으로 커밋되므로 별도의 `acknowledge()` 호출 불필요

## manual
### 개념
- acknowledge()가 여러 번 호출되어도, 실제 커밋은 다음 poll()에서 일괄로 처리된다.
- `acknowledge()` 호출 즉시가 아니라, 다음 poll() 시점에 batch로 커밋
### MANUAL 모드에서의 커밋
- `acknowledge()`가 호출되면 오프셋이 큐에 적재된다.
- 리스너 스레드가 배치 처리를 끝내고 다음 poll()을 호출하기 직전에, 큐에 쌓인 모든 오프셋을 한꺼번에 `commitSync()`(또는 `commitAsync()`)로 브로커에 전송한다.
- 현재 배치가 완전히 끝난 뒤에야 실제 커밋이 일어나며, 그 시점이 `다음 poll()`이다.
## 특징
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
## acknowledge()를 호출하지 않아도 내부적으로는 offset이 증가하는 문제
### 원인
- Spring Kafka에서는 메시지 처리와 offset commit이 별개의 과정으로 이루어진다.
- `MANUAL` ack 모드에서도 다음과 같은 동작이 발생합니다:
### 메시지 처리 위치 vs Commit된 Offset
- 처리 중인 offset: Kafka 컨슈머가 현재 읽어서 처리하고 있는 메시지의 위치
- Commit된 offset: 실제로 Kafka 브로커에 저장된 마지막으로 성공적으로 처리 완료된 위치
