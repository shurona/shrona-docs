---
tags:
  - sentinel
publish: false
---
Sentinel에 대한 자세한 내용은 해당 링크로 확인할 수 있다. [[Redis-Sentinel]]
# Spring에서 Redis Sentinel 설정 방법
Redis Config를 설정할 때 아래와 같이 Connection Factory를 설정해주면 된다.
```Java
@Bean
public RedisConnectionFactory lettuceConnectionFactory() {
  RedisSentinelConfiguration sentinelConfig = new RedisSentinelConfiguration()
  .master("mymaster")
  .sentinel("127.0.0.1", 26379)
  .sentinel("127.0.0.1", 26380);
  return new LettuceConnectionFactory(sentinelConfig);
}
```


## Sentinel에서 Docker 내부 호스트 IP 전달해주는 문제
[참고 블로그](https://blog.xavierz.dev/blog/posts/docker-redis-sentinel)


