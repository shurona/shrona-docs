---
tags:
  - transactioin
  - TransactionSynchronizationManager
---
# 발생한 문제
트랜잭션 커밋과 관계없이 TaskScheduler를 통해 작업을 예약한 결과,  커밋이 완료되기 전에 스케줄된 Task가 실행되어 DB에 저장되지 않은 데이터를 조회하려다 실패하는 문제가 발생했습니다.
# 문제 되는 코드
- 아래에서 registerTaskSchedule를 사용해서 비동기로 동작을 등록하였는데 messageLog가 저장되기 전에 읽어서 읽히는 메시지의 row가 0인 문제가 발생하였습니다.
```Java
@Transactional  
public List<MessageLog> createMessageAllGroup(Channel channel,  ...) {  

	// 로직들이 실행된다.
	...

    // 메시지를 저장한다.
    List<MessageLog> messageLogList = messageLogRepository.saveAll(groupInfo.stream()  
        .map(g -> createMessageLogForGroup(g, channel, typeInfo.get(),  
            reserveTime, content, exceptLineIds))  
        .toList());  
  
    messageUtils.registerTaskSchedule(messageLogList, reserveTime);  
  
    return messageLogList;  
}
```
# 해결 과정
- TransactionSynchronizationManager.registerSynchronization + afterCommit을 사용해서
  트랜잭션 커밋이 완료된 후에 TaskScheduler 등록이 될 수 있도록 설정하였다.
## TransactionSynchronizationManager란
- 현재 스레드에 바인딩된 트랜잭션 리소스(DB 커넥션, 세션 등)와 트랜잭션에 연관된 동기화 콜백(후처리 작업 등)을 관리하는 ThreadLocal 기반 유틸리티
### 대표 메소드
- 트랜잭션 리소스 등록/조회
```Java
TransactionSynchronizationManager.bindResource(DataSource, Connection);
TransactionSynchronizationManager.getResource(DataSource);
TransactionSynchronizationManager.unbindResource(DataSource);
```
- 트랜잭션 동기화 콜백 관리
```Java
TransactionSynchronizationManager.registerSynchronization
	(TransactionSynchronization synchronization);
```
- 트랜잭션 상태 조회
```Java
TransactionSynchronizationManager.isActualTransactionActive();
TransactionSynchronizationManager.isSynchronizationActive();
TransactionSynchronizationManager.getCurrentTransactionName();
```
## 예시 코드
```Java
// commit이 된 이후에 실행을 한다.  
TransactionSynchronizationManager.registerSynchronization(  
    new TransactionSynchronization() {  
        @Override  
        public void afterCommit() {  
            messageUtils.registerTaskSchedule(messageLogList, reserveTime);  
        }  
    }  
);
```
# 다른 방법
## @TransactionalEventListener
트랜잭션 이벤트 리스너 — 스프링 트랜잭션의 상태에 따라 특정 이벤트를 처리하는 리스너.
- 동기적으로 동작한다.
### 예시 코드
- 이벤트 클래스
```Java
public class MessageLogCreatedEvent {
    private final List<MessageLog> logs;
    private final LocalDateTime reserveTime;

    // 생성자 + Getter
}
```
- 이벤트 발행
```Java
@Transactional
public void createMessageLog(...) {
    messageLogRepository.saveAll(...);

    eventPublisher.publishEvent(new MessageLogCreatedEvent(logs, reserveTime));
}
```
- 이벤트 리스너
	- 이벤트 객체의 타입 및 파라미터를 기준으로 이벤트를 매칭한다.
```Java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handleMessageLogCreatedEvent(MessageLogCreatedEvent event) {
    messageUtils.registerTaskSchedule(event.getLogs(), event.getReserveTime());
}p
```
## @Async
스프링의 비동기를 진행하는 어노테이션
- 메서드 호출을 별도의 스레드에서 비동기적으로 실행.
- 기본적으로 Spring이 제공하는 TaskExecutor를 사용.
- 메인 쓰레드를 차단하지 않고 비동기로 처리할 수 있다는 장점이 있다.
### 예시 코드
- 스프링 비동기 활성화
```Java
@EnableAsync
@Configuration
public class AsyncConfig {
}
```
- Transaction commit이 끝난 후 비동기 적으로 동작한다.
```Java
@Async
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handleMessageLogCreatedEvent(MessageLogCreatedEvent event) {
    messageUtils.registerTaskSchedule(event.getLogs(), event.getReserveTime());
}
```
### taskExecutor 설정예시
```Java
@Bean(name = "taskExecutor")
public Executor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(10);
    executor.setMaxPoolSize(50);
    executor.setQueueCapacity(100);
    executor.setThreadNamePrefix("async-executor-");
    executor.initialize();
    return executor;
}
```