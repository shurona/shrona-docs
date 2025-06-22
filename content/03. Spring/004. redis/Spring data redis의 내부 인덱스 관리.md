---
tags:
  - spring/redis
  - index
---
# Redis에 Index를 사용할 경우 동작 방식
## 예시 코드 
```Java
@RedisHash(value = "token", timeToLive = (REFRESH_TOKEN_TIME / 1000))  
@NoArgsConstructor  
@AllArgsConstructor   
@Getter  
public class RefreshToken {  
  
    @Id  
    private Long userId;  
  
    @Indexed  
    private String token;  
  
    private LocalDateTime createdAt;  
}
```

## 실제 데이터 저장
- `RedisHash`를 사용하면 엔티티는 Redis 해시 구조로 저장이 된다.
- `token:{userId}`
```
HSET token:{userId}
  userId       {userId}
  token        {token값}
  createdAt    {생성일시}
```
## 인덱스 키
- `token:token:{token 데이터}`
- `@Indexed`가 붙은 프로퍼티에 대해서 값이 {token}인 엔티티 ID를 조회 할 수 있게 Set을 만들어 둔다.
```
SADD token:token:{token} {userId}
```
- `SMEMBERS token:token:{userId}`방식으로 유저 아이디를 빠르게 찾을 수 있다.
## 역색인 관리
- `token:{userId}:idx`
- 엔티티를 업데이트하거나 삭제할 때 어떤 인덱스(Set)에 있는 지 확인을 위한 정보
```
SADD token:{userId}:idx token:token:{token}
```
- 업데이트 시에는 이 `:idx`를 읽어와서 아래의 인덱스를 먼저 제거한 뒤 새 값으로 다시 `SADD`한다.
  `SREM token:token:OLD_TOKEN_VALUE {userId
## 발생한 문제
### TTL로 만료가 되거나 직접 Hash 데이터를 삭제한 경우 인덱싱 관련 데이터가 남아 있음
#### 원인
- `RedisHash`옵션은 엔티티의 메인 엔티티의 메인 해시에만 TTL을 설정한다.
- `@Indexed`가 붙은 token 필드를 위한 색인 Set과 역색인 관리를 위한 Set은 TTL이 없어서 삭제가 일어나지 않고 남아 있게 된다.
- 나중에 같은 userId로 새 RefreshToken을 저장하면 Spring Data Redis가 다시 인덱스 키를 만들기 때문에 기존 색인 데이터 위에 계속 쌓여서 쓸모 없는 키들이 늘어나게 된다.
#### 해결방법
- 색인 키에도 TTL을 설정하기
```
// 예시: RedisTemplate 사용
redisTemplate.expire("token:"+userId, REFRESH_TOKEN_TIME, TimeUnit.SECONDS);
redisTemplate.expire("token:"+userId+":idx", REFRESH_TOKEN_TIME, TimeUnit.SECONDS);
// 그리고 token:token:{token값} 세트에도 동일하게 expire 처리한다.
```
- Redis Keyspace Notifications를 활성화 해서 이벤트를 받아서 처리한다.
```Java
@Component
public class RedisExpiredListener
   extends KeyExpirationEventMessageListener {

  public RedisExpiredListener(RedisMessageListenerContainer c) { super(c); }

  @Override
  public void onMessage(Message msg, byte[] pattern) {
    String expiredKey = msg.toString();  // 예: token:123
    if (expiredKey.startsWith("token:") && !expiredKey.contains(":token:")) {
      // 1) idx 세트(token:123:idx)에서 옛 인덱스 키들을 꺼내
      // 2) 해당 token:token:oldValue 키들에서 123 제거
      // 3) token:123:idx 키 삭제
    }
  }
}
```