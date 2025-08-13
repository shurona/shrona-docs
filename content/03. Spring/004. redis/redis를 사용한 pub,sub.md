---
tags:
  - redis/pubsub
---
# Redis를 사용한 Pub/Sub 구조
## 용도
- 실시간 알림 시스템
- 실시간 채팅 시스템
- 분산 시스템 동기화
## 기본 설정
- `RedisMessageListenerContainer`는 Redis 채널을 모니터링하며 메시지 수신 시 등록된 리스너를 자동으로 호출하는 메소드
```Java
@Bean
public RedisMessageListenerContainer redisMessageListenerContainer(
        RedisConnectionFactory connectionFactory,
        MessageListenerAdapter listenerAdapter,
        ChannelTopic channelTopic
) {
    RedisMessageListenerContainer container = 
						    new RedisMessageListenerContainer();
    container.setConnectionFactory(connectionFactory);
    container.addMessageListener(listenerAdapter, channelTopic);
    return container;
}
```
## 메시지 처리 방식
### MessageListenerAdapter 방식
- 어댑터 설정
```Java
@Bean
public MessageListenerAdapter listenerAdapter(RedisSubscriber subscriber) {
    return new MessageListenerAdapter(subscriber, "onMessage");
}
```
- 구현체 설정
```Java
@Component
public class RedisSubscriber {

	private final ObjectMapper objectMapper;  
	private final RedisTemplate redisTemplate;  
    
    public void onMessage(String message, String channel) {
        log.info("채널: {}, 메시지: {}", channel, message);
        processMessage(message);
    }
}
```
### MessageListener 직접 구현 방식
- 어댑터에 설정
```Java
@Bean
public RedisMessageListenerContainer container(
        RedisConnectionFactory factory,
        RedisSubscriber subscriber) {
    
    RedisMessageListenerContainer container = 
						    new RedisMessageListenerContainer();
    container.setConnectionFactory(factory);
    container.addMessageListener(subscriber, new ChannelTopic("notification"));
    return container;
}
```
- 구현체 설정
```Java
@Component
public class RedisSubscriber implements MessageListener {
    
    @Override
    public void onMessage(Message message, byte[] pattern) {
        String channel = new String(message.getChannel());
        String publishMessage = (String) redisTemplate.getStringSerializer()
            .deserialize(message.getBody());
        
        log.info("채널: {}, 메시지: {}", channel, publishMessage);
        
        // 패턴 정보 활용 (패턴 구독 시)
        if (pattern != null && pattern.length > 0) {
            String patternStr = new String(pattern);
            log.info("매칭된 패턴: {}", patternStr);
            routeByPattern(patternStr, channel, publishMessage);
        }
    }
}
```
## 채널 구독 방식
### 특정 채널 구독
```Java
// 설정
container.addMessageListener(listener, new ChannelTopic("user.notification"));

// 메시지 발행
redisTemplate.convertAndSend("user.notification", "새로운 알림");
```
### 패턴 방식 구독
- 패턴을 추가하는 방식
```Java
// 설정 - 여러 채널을 와일드카드로 구독
container.addMessageListener(listener, new PatternTopic("user.*"));

// 메시지 발행 - 모두 수신됨
redisTemplate.convertAndSend("user.login", "로그인 알림");
redisTemplate.convertAndSend("user.logout", "로그아웃 알림");
redisTemplate.convertAndSend("user.notification", "일반 알림");
```
- 패턴 구독에서 pattern 파라미터 사용
```Java
public void onMessage(Message message, byte[] pattern) {
    String channel = new String(message.getChannel());
    String patternStr = pattern != null ? new String(pattern) : "직접 구독";
    
    log.info("패턴: {}, 실제 채널: {}", patternStr, channel);
    
    // 패턴별 분기 처리
    if ("user.*".equals(patternStr)) {
        handleUserEvent(channel, message);
    } else if ("order.*".equals(patternStr)) {
        handleOrderEvent(channel, message);
    }
}
```
