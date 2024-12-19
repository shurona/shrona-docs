---
tags:
  - spring
  - test/testcontainer
---

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

## Kafka 연결
위의 Redis는 일반적인 TestContainer를 이용해서 연결하는 것을 알아보았고 이번에는 각각 컨테이터에 특화된 KafkaContainer를 사용해서 import하는 법을 알아보았다.

### 설정 순서
테스트를 할 class위에 테스트 컨테이너를 사용하기 위해서 어노테이션 선언을 해준다.   
테스트 클래스의 생명 주기 동안 컨테이너를 유지합니다.
```Java
@Testcontainers
public class CouponUserServiceImpl implements CouponUserService {
	...
}
```

아래의 코드를 이용해서 이미지를 띄우게 된다. 위에서 선언한 Container 와는 다르게  
포트는 따로 등록해줄 필요가 없이 자체적으로 처리를 하게 된다.   
로컬에서 띄울 경우 기존에 로컬 서비스와 포트가 충돌 될 수도 있으니 랜덤으로 뜰 수 있도록 설정한다. 
```Java
static final KafkaContainer kafka = new KafkaContainer(  
	    DockerImageName.parse(KAFKA_IMAGE));
```

테스트 용 Kafka 포트는 랜덤으로 실행이 되므로 위에서 생성된 Kafka Docker 이미지로 부터 메타데이터 정보를 받아서 동적으로 Property를 넣어준다.
```Java
@DynamicPropertySource  
	static void setKafkaProperties(DynamicPropertyRegistry registry) {  
	    registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);  
	}
```

### 연결 기본 코드
```Java
...
// 여기서 import 위치 조심
import org.testcontainers.containers.KafkaContainer;
...

@Testcontainers
class CouponUserServiceImplTest {

	private static final String KAFKA_IMAGE = "confluentinc/cp-kafka:7.7.1";  
  
	//kafka Container  
	@Container  
	static final KafkaContainer kafka = new KafkaContainer(  
	    DockerImageName.parse(KAFKA_IMAGE));


	@DynamicPropertySource  
	static void setKafkaProperties(DynamicPropertyRegistry registry) {  
	    System.out.println("?? 여기는 실행되나요? " + kafka.getBootstrapServers());  
	    registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);  
	}

	// 아래는 테스트 로직
	...
}
```

### 여기서 삽질 했던 부분
#### 발생했던 에러
초반에 아래 부분 같이 KafkaContainer를 설정했었다.
```Java
@Container  
	static final KafkaContainer kafka = new KafkaContainer(  
	    DockerImageName.parse(KAFKA_IMAGE));
```
그랬더니 아래와 같은 에러가 발생했었다.
`Failed to verify that image 'confluentinc/cp-kafka:7.4.0' is a compatible substitute for 'apache/kafka'.`
이 문제를 해결하기 위해서 [TestContainer Java](https://java.testcontainers.org/modules/kafka/)를 확인했었고 `ConfluentKafkaContainer`의 클래스를 사용했어야 했으나 현재 라이브러리에는 존재하지 않는 것을 확인하였다.
그래서 해당 공식문서에 연결된 블로그를 확인해보니 `KafkaContainer`를 사용해서 예제를 만드는 것을 확인하였다. 
그래도 로컬에서는 같은 에러가 발생했었고 원인을 찾아보았다.

#### 해결
간단한 문제였는데 아래와 같이 Import되는 위치를 변경해주면 되었다.
```Java
// (잘못된 위치)
import org.testcontainers.kafka.KafkaContainer;
=>
// (여기를 사용하자)
import org.testcontainers.containers.KafkaContainer;
```



### 참고 블로그
[test container예제](https://testcontainers.com/guides/testing-micronaut-kafka-listener-using-testcontainers/#_write_test_for_kafka_listener)