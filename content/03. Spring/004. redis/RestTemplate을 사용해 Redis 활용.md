## RestTemplate
Key를 직접 설정하고 자료형을 선택해서 세밀하게 데이터 타입을 다룸으로써 Redis활용이 가능하다.

### Config
#### Redis Connection Factory
```Java
@Bean  
public RedisConnectionFactory connectionFactory() {  
  
    RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();  
  
    config.setHostName(redisProperty.host());  
    config.setUsername(redisProperty.username());  
    config.setPort(redisProperty.port());  
    config.setPassword(redisProperty.password());  
  
    return new LettuceConnectionFactory(config);  
}
```
#### Redis Template
아래와 같이 Object로 지정을 할 수 있고
```Java
@Bean  
public RedisTemplate<String, Object> couponSetRedisTemplate() {  
  
    RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();  
    redisTemplate.setConnectionFactory(connectionFactory());  
  
    // key-value  
    redisTemplate.setKeySerializer(new StringRedisSerializer());  
    redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());  
  
    // hash  
    redisTemplate.setHashKeySerializer(new StringRedisSerializer());  
    redisTemplate.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());  
  
    return redisTemplate;  
}
```
만약 특정 타입을 위해서 RedisTemplate을 설정을 한다면 아래와 같이 설정을 해줄 수도 있다.
```Java
@Bean  
public RedisTemplate<String, Integer> integerRedisTemplate() {  
  
    RedisTemplate<String, Integer> redisTemplate = new RedisTemplate<>();  
    redisTemplate.setConnectionFactory(connectionFactory());  
  
    // key-value  
    redisTemplate.setKeySerializer(new StringRedisSerializer());  
    redisTemplate.setValueSerializer(new GenericToStringSerializer<>(Integer.class));  
  
    // hash  
    redisTemplate.setHashKeySerializer(new StringRedisSerializer());  
    redisTemplate.setHashValueSerializer(new GenericToStringSerializer<>(Integer.class));  
  
    return redisTemplate;  
}
```

## 주로 사용하는 Method
- opsForValue()
	- 문자열 작업
- opsForHash()
	- 해시 작업
- opsForList()
	- 배열(리스트) 작업
- opsForSet()
	- 집합(중복 처리) 작업
- opsForZSet()
	- 순서를 보장한 집합 작업
	- rank를 사용한 우선순위
- opsForGeo()
	- 지리 처리
- 이외 메서드
	- delete
		- 키 삭제
	- expire
		- 특정 시간 뒤 만료되도록 설정
### 예시
Spring에서는 위에서 Config로 설정한 템플릿을 아래와 같이 주입해 줄 수 있다
```Java
private final RedisTemplate<String, Object> redisTemplate;  
private final RedisTemplate<String, Integer> redisIntegerTemplate;
```

이후 원하는 함수를 사용해서 Redis에 데이터를 저장 할 수 있다
```Java
redisTemplate
	.opsForZSet()
	.add(redisKey, userId, System.currentTimeMillis());
```

```Java
String redisKey = convertCouponIsOpenRedisKey(couponId);  
  
ValueOperations<String, Integer> opsForValue =  
    redisIntegerTemplate.opsForValue();  
  
// 상태 지정  
opsForValue.set(redisKey, wishStatus, durationHour, TimeUnit.HOURS);
```