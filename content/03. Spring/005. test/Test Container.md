## 테스트 container의 사용 이유

테스트 환경에서 redis와 같은 외부 서비스를 CI 환경에서 테스트 하기 위해서는 CI 환경에서 Redis를 띄우거나 Github action marketplace에서 redis를 추가해서 실행을 할 수 있다.

하지만 추후에 DB나 Kafka와 같이 다른 서비스를 사용해야 할 경우 모든 서비스를 임포트 하기 보다는 Spring에서 공식적으로 제공하는 테스트용 컨테이너를 설정해서 사용하면 CI 뿐만 아니라 다른 로컬 환경에서 테스트 할 때도 별도 세팅을 변경할 필요 없이 사용할 수 있을 것이라고 판단해서 진행하였다.

## 초기 설정
[start.spring.io](http://start.spring.io/) 에서 testContainer의 build.gradle 설정을 갖고 온다.
```gradle
// 테스트 container 
testImplementation 'org.springframework.boot:spring-boot-testcontainers'
```

## 개념 설명
[Spring TestContiainer 문서](https://docs.spring.io/spring-boot/reference/testing/testcontainers.html)
Testcontainers 라이브러리는 Docker 컨테이너 안에서 서비스가 동작하고 다루는 방식을 제공한다.
JUnit과 통합되어 있으며 테스트가 동작하기 전에 컨테이너가 실행이 된다.

## Config 설정
먼저 레디스 Container를 실행시켜 준 다음에 실행된 redis로 부터 Host와 Port 정보를 갖고 와서 property로 등록해준다.
```Java
@TestConfiguration
public class TestContainerConfig implements BeforeAllCallback {

    public final String REDIS_IMAGE = "redis:7.4.0-alpine";
    public final Integer port = 6379;


    @Override
    public void beforeAll(ExtensionContext context) throws Exception {
        GenericContainer<?> redis = new GenericContainer(
            DockerImageName.parse(REDIS_IMAGE)).withExposedPorts(port);

        redis.start();
        System.setProperty("spring.data.redis.host", redis.getHost());
        System.setProperty("spring.data.redis.port", String.valueOf(redis.getMappedPort(port)));

    }
}
```

## 발생했던 문제점
### Nested 마다 Redis 컨테이너 설정
테스트를 돌리는 과정 중에서 Container의 라이프 사이클에 대해서 궁금증이 들었었다.
그래서 Container가 생성되는 모습을 확인해 봤더니  Nested 갯수 마다 컨테이너가 실행이 되는 것을 확인할 수 있었다.
`@Nested`가 2개인 Test를 돌렸을 때 발생하는 컨테이너 예시

| name               | image                     |
| ------------------ | ------------------------- |
| testcontainer-ryuk | testcontainers/ryuk:0.0.0 |
| redis-name-1       | redis:이미지                 |
| redis-name-2       | redis:이미지                 |
테스트 마다 격리된 구축 환경을 구축하는 것이 맞기 때문에 위와 같이 진행되는 것이 정상적인 Flow지만 만약 리소스가 부족한 환경이라면 아래와 같이 하나의 Redis에서 띄우는 것도 가능하다
GenericContainer를 생성한 다음에 아래와 같이 이미 존재한 경우에는 띄우지 않는다.
```Java
public static final String REDIS_IMAGE = "redis:7.4.0-alpine";
public static final Integer port = 6379;
public static GenericContainer<?> redis;

@Override
public void beforeAll(ExtensionContext context) throws Exception {
    if (redis == null) {
        redis = new GenericContainer(
            DockerImageName.parse(REDIS_IMAGE)).withExposedPorts(port).withReuse(true);

        redis.start();
        System.setProperty("spring.data.redis.host", redis.getHost());
        System.setProperty("spring.data.redis.port", String.valueOf(redis.getMappedPort(port)));
    }
}
```
