# 특징

## Eureka Server

- Eureka Server는 서비스 등록해주는 역할을 하며, 모든 마이크로서비스가 자신을 등록하고 다른 서비스의 위치를 검색할 수 있도록 한다.
- Eureka Server에 서비스가 등록되면, 다른 서비스들이 이 정보를 기반으로 통신할 수 있다.
- 일반적으로 마이크로서비스 애플리케이션에서 하나 이상의 Eureka Server 인스턴스를 실행하여 고가용성을 확보한다.

## Eureka Client

- 각 마이크로서비스는 Eureka Client로 Eureka Server에 자신을 등록하고 주기적으로 상태 정보를 전송한다.
- Eureka Client는 Eureka Server로부터 다른 서비스의 정보를 가져오고, 이를 서비스 디스커버리에 활용합니다.

### 장점
- 자동화된 서비스 등록
	- Spring Boot 애플리케이션이 시작될 때 Eureka Server에 자동으로 등록되어, 서비스 디스커버리가 자동화된다.
-  클라이언트 측 로드 밸런싱
	- Spring Cloud Netflix Ribbon을 함께 사용하여 클라이언트 측 로드 밸런싱을 쉽게 구현할 수 있다.
- 구성 관리와 통합
	- Spring Cloud Config, Spring Boot Actuator와 함께 Eureka를 사용하여 마이크로서비스 구성 및 모니터링을 통합할 수 있다.
- 손쉬운 설정
	- application.yml 또는 application.properties에서 간단히 Eureka 설정을 추가하여 마이크로서비스가 쉽게 Eureka에 연결될 수 있다.

# 설정 과정

## Eureka Server 설정

#### Eurkea의 서버로 지정된 곳에서는 서버라고 지칭해준다.
```Java
@EnableEurekaServer
@SpringBootApplication
public class ServerApplication() {
	//
}
```

#### application.properties에서 설정을 해준다.
```properties
// 자기 자신을 서비스로 등록하지 않음
eureka.client.register-with-eureka=false

// 마이크로 서비스 인스턴스를 로컬에 캐시할 것인가
eureka.client.fetch-registery=false

// eureka의 서버 위치를 알 수 있게 한다.
eureka.client.service-url.defaultZone=http://localhost:port/eureka/
```

## Eureka client 설정
#### application.properties에서 Eureka의 서버 위치를 알려줘야 한다.
```properties
eureka.client.service-url.defaultZone:http://localhost:port/eureka/

// 인스턴스의 고유 아이디를 지정할 수 있다.
eureka.instance.instance-id={spring.cloud.client.hostname}:${spring.application.instance_id:${random.value}}
```
#### 서버의 이름은 application name을 따라간다
```yaml
spring:
	application:
		name={서버이름}
```

