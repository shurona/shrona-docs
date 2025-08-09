---
tags:
  - kafka/retry
---
# Retry 정책
## 고정 BackOff (Fixed BackOff)
- **개념**: 재시도 간격을 일정하게 유지하는 방식
- **특징**: 매번 동일한 시간 간격으로 재시도
- **장점**: 예측 가능한 패턴, 구현이 간단
- **단점**: 서버 부하가 지속될 수 있음
```java
// 1초 간격, 최대 3회 재시도
FixedBackOff backOff = new FixedBackOff(1000L, 3L);
```

## 증가 BackOff (Exponential BackOff)  
- **개념**: 재시도할 때마다 대기 시간을 증가시키는 방식
- **특징**: 첫 시도 후 시간을 점진적으로 늘려감 (예: 1초 → 2초 → 4초)
- **장점**: 서버 부하를 점진적으로 완화, 일시적 장애에 효과적
- **단점**: 복구 시간이 길어질 수 있음
```java
// 2초 시작, 2배씩 증가, 최대 10초
RetryTopicConfigurationBuilder
    .newInstance()
    .exponentialBackoff(2000, 2.0, 10000)
```

## 랜덤 BackOff (Random BackOff)
- **개념**: 재시도 간격에 임의성을 추가하는 방식
- **특징**: 기본 간격에 랜덤 요소를 더해 분산시킴
- **장점**: 동시 재시도로 인한 서버 부하 집중 방지 (Thundering Herd 방지)
- **단점**: 예측하기 어려운 패턴
```java
// 기본 간격 + 랜덤 요소 추가
ExponentialBackOff backOff = new ExponentialBackOff(1000L, 2.0);
backOff.setRandomizeMultiplier(true); // 랜덤 요소 활성화
```

## 재시도 정책 선택 가이드
- **고정 BackOff**: 빠른 복구가 예상되는 일시적 네트워크 오류
- **증가 BackOff**: 서버 과부하나 외부 시스템 장애 시
- **랜덤 BackOff**: 대량의 동시 요청이 예상되는 환경에서


# 리트라이 핸들러 설정
## DefaultErrorHandler
- 카프카 내부에서 토픽 생성 없이 컨슈머 스레드에서 재시도 한다.
```Java
@Configuration
public class BlockingRetryConfig {
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, ChatMessage> 
           kafkaListenerContainerFactory (
           ConsumerFactory<String, String> consumerFactory,
           KafkaTemplate<String, String> kafkaTemplate
   ) {
        ConcurrentKafkaListenerContainerFactory<String, String> factory =  
		    new ConcurrentKafkaListenerContainerFactory<>();  
		factory.setConsumerFactory(consumerFactory);
        factory.getContainerProperties().setAckMode(AckMode.MANUAL_IMMEDIATE);
        
        // DLT 전송을 위한 Recoverer
        DeadLetterPublishingRecoverer recoverer = 
            new DeadLetterPublishingRecoverer(kafkaTemplate,
                (record, exception) -> {
                    if (exception instanceof ValidationException) {
                        return new TopicPartition("chat-validation-errors", 0);
                    }
                    return new TopicPartition(record.topic() + ".dlt", record.partition());
                });
        
        // 블로킹 재시도: 1초 간격, 최대 2회
        DefaultErrorHandler errorHandler = 
            new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 2L));
        
        // 재시도 가능한 예외 지정
        errorHandler.addRetryableExceptions(RecoverableException.class);
        errorHandler.addNotRetryableExceptions(ValidationException.class);
        
        factory.setCommonErrorHandler(errorHandler);
        return factory;
    }
}

```
## Retry Topic
- 별도의 재시도 토픽을 생성해서 처리한다.
```Java
@Configuration
@EnableKafkaRetryTopic
public class NonBlockingRetryConfig {
    
    @Bean
    public RetryTopicConfiguration chatRetryConfiguration(
            KafkaTemplate<String, String> kafkaTemplate) {
        
        return RetryTopicConfigurationBuilder
            .newInstance()
            .maxAttempts(3)
            .exponentialBackoff(2000, 2.0, 10000) // 2초 시작, 2배씩 증가, 최대 10초
            .dltSuffix(".dlt") // 지정된 Topic에서 dlt 추가
            .retryOn(RecoverableException.class) // 재시도 할 Exception
            .notRetryOn(ValidationException.class) // 재시도 하지 않을 Exception
            .includeTopics("chat-topic") // 적용할 토픽
            .autoCreateTopics(true) // 자동 토픽 생성
            .numPartitions(3)
            .replicationFactor(2)
            .create(kafkaTemplate);
    }
}
```

