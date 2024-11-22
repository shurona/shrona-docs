# LogStash를 Spring에 적용
## 과정
### build.gradle에 추가
```gradle
// # Logback <-> Logstash 연동을 위함 
implementation 'net.logstash.logback:logstash-logback-encoder:8.0'
```
### Spring 에서 LogStash 설정하기
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<configuration scan="true" scanPeriod="30 seconds">
  <!-- 콘솔에 로그 출력 형식 지정 -->
  <appender class="ch.qos.logback.core.ConsoleAppender" name="CONSOLE">
    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
      <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} springboot-elk [%thread] %-5level %logger{36} - %msg%n
      </pattern>
    </encoder>
  </appender>

  <!-- Logstash로 로그 전송 -->
  <appender class="net.logstash.logback.appender.LogstashTcpSocketAppender" name="LOGSTASH">
    <destination>127.0.0.1:50000</destination>
    <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
      <providers>
        <logLevel/>
        <loggerName/>
        <message/>
        <stackTrace/>
        <threadName/>
        <timestamp>
          <timeZone>UTC</timeZone>
        </timestamp>
      </providers>
    </encoder>

    <param name="Encoding" value="UTF-8"/>
  </appender>

  <root level="INFO">
    <appender-ref ref="CONSOLE"/>
    <appender-ref ref="LOGSTASH"/>
  </root>

</configuration>
```
LogStash를 적용하는 XML 파일의 정보이다.
#### 스캔 설정
`<configuration scan="true" scanPeriod="30 seconds">`
LogBack이 XML 설정파일의 변경 여부를 감시하는 설정이다. 30초 마다 스캔으로 설정을 함으로써 XML이 변경될 때마다 반영이 되도록 한다.

#### 콘솔에 로그 출력 형식 지정
`class="ch.qos.logback.core.ConsoleAppender"`
콘솔로 로그를 출력하겠다는 의미이다.
`class="ch.qos.logback.classic.encoder.PatternLayoutEncoder"`
encoder는 로그 메세지의 포맷을 정리하며 아래의 설정은 PatternLayout형식으로 로그를 인코딩하는 것을 의미한다.
아래와 같은 패턴으로 로그가 출력 됨
```xml
<pattern> 
// 연월일                     고정값(검색용)  쓰레드이름 로그 레벨 로거이름 최대 36자까지
%d{yyyy-MM-dd HH:mm:ss.SSS} springboot-elk [%thread] %-5level %logger{36} 

 메세지 표시
- %msg%n </pattern>
```

#### 로그를 JSON 형식으로 LogStash로 전송
`class="net.logstash.logback.appender.LogstashTcpSocketAppender" name="LOGSTASH"`
로그를 TCP 소켓을 이용해서 전송한다는 것을 의미한다.

`<destination>127.0.0.1:50000</destination>`
전송이 될 위치를 지정하는 부분이다.

```xml
<encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
  <providers>
    <logLevel/>
    <loggerName/>
    <message/>
    <stackTrace/>
    <threadName/>
    <timestamp>
      <timeZone>UTC</timeZone>
    </timestamp>
  </providers>
</encoder>
```
provider는 로그에 포함될 항목들을 정의하며 
Log데이터가 위와 같은 형식의 JSON으로 파싱되도록 지정할 수 있다.

`<param name="Encoding" value="UTF-8"/>`
전송될 메시지의 String Format을 지정해 줄 수 있다.
#### LogBack의 로그 level을 지정
```xml
<root level="INFO">
  <appender-ref ref="CONSOLE"/>
  <appender-ref ref="LOGSTASH"/>
</root>
```
위의 예제는 INFO 부터 기록한다는 의미이다.
> **로그 레벨**
> TRACE    DEBUG   INFO   WARN   ERROR

## 서비스 마다 다른 인덱스를 위한 MDC 필드 사용
모든 서비스가 같은 형식의 로그를 갖고 있으면 Elastic 에서 검색을 할 때 원하는 서비스의 로그를 찾는데 어려움이 있다.
이를 해결하기 위해서 서비스 모듈마다 로그에 어떤 서비스에서 로그가 발생했는지 표시해주기 위해서 서비스 고유 아이디를 로그에 추가해준다.
다만 여기서 단순히 String 값을 전달되는 로그로 추가 하는 것이 아니라 Elastic Search의 KeyToken으로 추가를 해주기 위해서는 LogStash에서 Elastic Cloud로 전달해 줄 때 구분할 수 있는 인덱싱 정보도 같이 전달해줘야 한다.
LogStash에서 서비스들을 구분하기 위해서 MDC 필드를 이용하기로 하였다.

>**MDC 필드란**
   MDC란 SLF4J 로깅 라이브러리에서 제공하는 기능으로 현재 쓰레드에 메타 정보를 넣고 관리하는 공간이다. MDC는 내부적으로 Map을 관리하고 있어서 Key-Value 형태로 값을 저장할 수 있다.

#### Filter를 사용해서 로그에 MDC 적용
```java
@Override
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
    throws IOException, ServletException {
    try {
        MDC.put("service", SERVICE_NAME);
        chain.doFilter(request, response);
    } finally {
        MDC.clear();
    }
}
```
위와 같이 Filter에서 Request가 발생할 때 service를 주입해준다.
다만 쓰레드가 재사용 될 때 데이터가 남아있을 수 있으므로 Request가 종료될 때에는 초기화 해줘야 한다.
이후 `logback-spring.xml`에서 LogStash로 로그 데이터를 전송해줄 때 mdc 정보도 같이 전달해준다.
```xml
<appender class="net.logstash.logback.appender.LogstashTcpSocketAppender" name="LOGSTASH">
    <destination>${logstashHost}:${logstashPort}</destination>
    <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
      <providers>
        <logLevel/>
        <loggerName/>
        <mdc/>
        ...
```

LogStash에서 넘어온 mdc 정보를 filter를 사용해서 조건문을 적용해 줄 수 있다.

이후 mutate를 사용해서 새로운 필드를 추가해준 다음에 output의 elasticSearch로 전달되는 index에 변수 값을 전달해준다.
>  **mutate의 기능**
> - add_field: 필드를 추가한다. 
> - remove_field: 필드를 제거한다. 
> - rename: 필드 이름을 변경한다.

```conf
filter {
  # 로그에 서비스 이름이 포함된 필드를 사용해 분류
  if [service] == "rider-service" {
    mutate { add_field => { "index_name" => "rider-service" } }
  } else if [service] == "coupon-service" {
    mutate { add_field => { "index_name" => "coupon-service" } }
  } else {
  	mutate { add_field => { "index_name" => "springboot-elk" } }
  }
}

index => "%{index_name}"
```

## logback-spring.xml Production과 Local에서 변수 분리
`logback-spring.xml` 에서 local과 production 환경에서 LogStash url하고 Log Level을 분리하고 싶었다. 
그래서 처음에는 `application.yml`과 같이 `${}` 표현식을 사용해서 간단히 해결 될 수 있을 것이라고 생각했으나 이런 접근으로는 데이터를 주입할 수가 없었다.
이를 해결하기 위해서 공식문서를 확인해 보니 XML Tag를 사용해서 변수를 주입해 줄 수가 있었다.
1. application.yml 파일에서 데이터 추가
아래와 같이 LogStash가 설정된 인스턴스의 URL을 설정한다.
```yaml
log:
  logstash:
    host: ${LOG_STASH_HOST}
    port: ${LOG_STASH_PORT}
```
2. `logback-spring.xml`에서 springProfile을 사용한다.
springProfile을 이용하면 xml파일에서 현재 profile에 맞춰서 설정을 적용해 줄 수 있다.
아래와 같이 `spring.profiles.active=dev` 인 경우에는 log level을 WARN으로 설정해준다.
```xml
<springProfile name="dev">
  <root level="WARN">
    <appender-ref ref="CONSOLE"/>
    <appender-ref ref="LOGSTASH"/>
  </root>
</springProfile>
```
3. `springProperty`를 사용해서 변수를 선언해준다.
springProperty 태그를 사용하면 Spring의 `application.yml`파일을 Logback 설정 파일에서 접근을 할 수 있게 해준다.
```xml
<springProperty defaultValue="127.0.0.1" name="logstashHost" scope="context" source="log.logstash.host"/>
```
source를 통해서 property를 지정한 다음에 변수 이름을 지정하고 scope를 context로 지정함으로 전역에서 접근할 수 있게 한다.
```xml
<appender class="net.logstash.logback.appender.LogstashTcpSocketAppender" name="LOGSTASH">
    <destination>${logstashHost}:${logstashPort}</destination>
    ...
```
위와 같이 사용할 수 있다.

## 참고 블로그
[How to set up Filebeat and LogStash with ElasticSearch](https://www.gosink.in/how-to-set-up-filebeat-and-logstash-with-elastic-search-and-elastic-cloud/)
[Spring 멀티쓰레드 환경에서 MDC를 사용해 요청 별로 식별가능한 로그 남기기](https://mangkyu.tistory.com/266)
[Logging :: Spring Boot](https://docs.spring.io/spring-boot/reference/features/logging.html#features.logging.logback-extensions.environment-properties)