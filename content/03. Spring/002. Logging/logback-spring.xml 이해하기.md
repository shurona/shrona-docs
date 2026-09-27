---
tags:
  - logging
  - logback
  - spring-boot
---
# logback-spring.xml 이해하기

> [[배포 후 로그 관리를 해보자]] 로드맵의 3단계

[[YAML로 로그 설정하기]]에서는 XML 없이 YAML만으로 레벨, 파일 출력, 패턴, 롤링을 설정했다. 이 문서는 그 YAML 설정이 안에서 어떻게 동작하는지, 그리고 YAML로 부족할 때 직접 쓰는 `logback-spring.xml`을 정리한다.

## YAML 설정의 정체: 기본 설정의 빈칸 채우기

Spring Boot는 로그 설정 파일이 없으면 기본 Logback 설정을 스스로 만들어 쓴다. 같은 내용이 XML로도 jar 안에 들어 있어서 직접 열어볼 수 있다. 아래는 Spring Boot 3.4.5 jar에 들어 있는 `file-appender.xml`이다. (일부 줄 생략, 오른쪽 주석은 대응하는 YAML 설정이고 `rollingpolicy`는 `logging.logback.rollingpolicy`를 줄여 적은 것)

```xml
<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        <level>${FILE_LOG_THRESHOLD}</level>                                    <!-- logging.threshold.file -->
    </filter>
    <encoder>
        <pattern>${FILE_LOG_PATTERN}</pattern>                                  <!-- logging.pattern.file -->
    </encoder>
    <file>${LOG_FILE}</file>                                                    <!-- logging.file.name -->
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>${LOGBACK_ROLLINGPOLICY_FILE_NAME_PATTERN:-${LOG_FILE}.%d{yyyy-MM-dd}.%i.gz}</fileNamePattern>
        <maxFileSize>${LOGBACK_ROLLINGPOLICY_MAX_FILE_SIZE:-10MB}</maxFileSize>   <!-- rollingpolicy.max-file-size -->
        <totalSizeCap>${LOGBACK_ROLLINGPOLICY_TOTAL_SIZE_CAP:-0}</totalSizeCap>  <!-- rollingpolicy.total-size-cap -->
        <maxHistory>${LOGBACK_ROLLINGPOLICY_MAX_HISTORY:-7}</maxHistory>          <!-- rollingpolicy.max-history -->
    </rollingPolicy>
</appender>
```

`${...}`는 변수 자리이다. Spring Boot는 시작할 때 YAML의 `logging.*` 값을 이 변수들에 넣어준다. 즉 YAML 설정은 이미 만들어진 설정의 빈칸을 채우는 것이다.

`${이름:-기본값}`은 값이 없으면 기본값을 쓰라는 뜻이다. YAML에 아무것도 적지 않으면 파일당 10MB, 보관 7일, 전체 용량 제한 없음(0)이 되는 것도 여기서 나온다. 2단계 실습에서 생긴 `mommytalk.log.2026-08-08.0.gz`라는 이름도 기본 패턴 `${LOG_FILE}.%d{yyyy-MM-dd}.%i.gz`에서 나온 것이다.

반대로 `logback-spring.xml`은 이 설정 자체를 직접 쓰는 것이다. 빈칸에 들어갈 값이 아니라 appender의 개수, 연결 방식, 필터 같은 구조를 바꿀 수 있다.

## logback.xml과 logback-spring.xml

두 파일 모두 `src/main/resources`에 두면 자동으로 읽힌다. 차이는 읽히는 시점이다.

| | logback.xml | logback-spring.xml |
|---|---|---|
| 읽히는 시점 | Spring 환경이 준비되기 전에 Logback이 먼저 읽는다 | Spring Boot가 프로필과 YAML 값을 준비한 뒤 읽는다 |
| Spring 전용 태그 | 쓸 수 없다 | `<springProfile>`, `<springProperty>`를 쓸 수 있다 |

Spring Boot 공식 문서도 가능하면 `-spring`이 붙은 파일을 쓰라고 권장한다. 이 문서도 `logback-spring.xml`을 기준으로 정리한다.

## 기본 구조

[[SLF4J와 Logback의 관계]]에서 본 개념 4가지가 그대로 XML 태그가 된다.

| 개념 | XML 태그 |
|---|---|
| Logger | `<logger name="...">`, 최상위는 `<root>` |
| Level | `level` 속성 |
| Appender | `<appender name="..." class="...">` |
| Encoder / Pattern | `<appender>` 안의 `<encoder><pattern>` |

가장 작은 설정은 이렇다.

```xml
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="com.shrona.mommytalk" level="DEBUG"/>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
    </root>
</configuration>
```

- `CONSOLE`이라는 appender를 만들고, 출력 형식을 `<pattern>`으로 정한다.
- `com.shrona.mommytalk` 패키지는 DEBUG, 나머지는 root의 INFO를 따른다.
- root에 `CONSOLE`을 연결했으므로 모든 로그가 콘솔로 나간다.

### appender는 연결해야 동작한다

`<appender>`는 정의만 해서는 아무 일도 하지 않는다. `<root>`나 `<logger>` 안에서 `<appender-ref>`로 연결해야 로그가 들어간다. appender를 추가했는데 파일이 생기지 않는다면 가장 먼저 확인할 부분이다.

### 로그는 부모의 appender로도 전달된다 (additivity)

레벨은 부모에서 자식으로 물려받는다. 반대로 로그는 자식 logger의 appender에 찍힌 다음 부모 logger의 appender로도 올라간다.

```xml
<!-- KAKAO_FILE appender 정의는 생략 -->
<logger name="com.shrona.mommytalk.kakao" level="INFO">
    <appender-ref ref="KAKAO_FILE"/>
</logger>

<root level="INFO">
    <appender-ref ref="CONSOLE"/>
</root>
```

이 설정에서 kakao 패키지의 로그는 `KAKAO_FILE`과 root의 `CONSOLE` 양쪽에 모두 찍힌다. 이 성질을 additivity라고 한다. kakao 로그를 전용 파일에만 남기려면 `additivity="false"`를 준다.

```xml
<logger name="com.shrona.mommytalk.kakao" level="INFO" additivity="false">
    <appender-ref ref="KAKAO_FILE"/>
</logger>
```

## 롤링 정책

`RollingFileAppender`는 롤링 정책으로 파일을 언제 자르고 얼마나 남길지 정한다. `max-history`와 `total-size-cap`이 무엇을 뜻하는지는 [[YAML로 로그 설정하기]]에서 실습과 함께 정리했으므로, 여기서는 XML에서 새로 알게 되는 것만 적는다.

### 정책 두 가지

| 정책 | 자르는 기준 | 파일 이름 패턴 예 |
|---|---|---|
| `TimeBasedRollingPolicy` | 날짜(시간) 주기 | `app.%d{yyyy-MM-dd}.log.gz` |
| `SizeAndTimeBasedRollingPolicy` | 날짜 주기 + 파일 크기(`maxFileSize`) | `app.%d{yyyy-MM-dd}.%i.log.gz` |

Spring Boot 기본 설정은 `SizeAndTimeBasedRollingPolicy`를 쓴다. 크기로도 자르기 때문에 같은 날 안에서 파일을 구분할 번호 `%i`가 반드시 필요하다. 2단계에서 본 `.0.gz`, `.1.gz`의 번호가 바로 `%i`이다.

### 주기는 파일 이름 패턴이 정한다

롤링 주기를 따로 정하는 설정은 없다. `%d{...}` 안의 가장 작은 단위가 주기가 된다.

- `%d{yyyy-MM-dd}`: 하루마다
- `%d{yyyy-MM-dd-HH}`: 1시간마다

그래서 `maxHistory`의 단위도 패턴을 따라간다. 하루 주기에서 `maxHistory 30`은 30일이지만 1시간 주기에서는 30시간이다. 파일 이름이 `.gz`나 `.zip`으로 끝나면 롤링할 때 자동으로 압축한다.

YAML에서도 `logging.logback.rollingpolicy.file-name-pattern`으로 이 패턴을 바꿀 수 있다.

### totalSizeCap은 maxHistory가 있어야 동작한다

`maxHistory` 없이 `totalSizeCap`만 쓰면 전체 용량 제한은 무시된다. Logback 1.5.18 코드에 `'maxHistory' is not set, ignoring 'totalSizeCap' option`이라는 경고 메시지가 들어 있다. YAML로 설정할 때는 기본 설정이 `maxHistory`를 항상 7로 채워주기 때문에 이 문제를 만날 일이 없고, XML을 직접 쓸 때만 조심하면 된다.

## XML을 만들면 YAML 설정은 어떻게 되나

`logback-spring.xml`을 만들면 Spring Boot는 기본 설정 대신 이 파일을 쓴다. 이때 YAML의 `logging.*` 설정은 두 가지로 나뉜다.

**계속 적용되는 것: `logging.level`**

Spring Boot는 XML을 읽은 다음 YAML의 레벨을 다시 적용한다. 그래서 같은 logger의 레벨이 XML과 YAML에 모두 있으면 YAML 값이 이긴다.

**XML이 참조할 때만 적용되는 것: 파일, 패턴, 롤링, threshold**

`logging.file.name`, `logging.pattern.*`, `logging.logback.rollingpolicy.*` 같은 값은 Spring Boot가 `${LOG_FILE}`, `${FILE_LOG_PATTERN}` 같은 변수로 넘겨줄 뿐이다. XML이 그 변수를 쓰지 않으면 아무 효과가 없다. 예를 들어 XML에 `<file>logs/app.log</file>`처럼 값을 직접 적으면 YAML의 `logging.file.name`은 무시된다.

### Spring Boot 기본 설정을 가져다 쓰기

기본 설정을 살리면서 필요한 것만 더하고 싶으면 jar 안의 기본 설정 파일을 include한다.

```xml
<configuration>
    <include resource="org/springframework/boot/logging/logback/base.xml"/>

    <!-- 기본 CONSOLE, FILE은 그대로 두고 필요한 appender만 여기에 추가 -->
</configuration>
```

`base.xml`은 기본 설정 전체(변수 기본값, CONSOLE, FILE, root INFO)를 한 번에 가져온다. 이렇게 include하면 YAML의 파일, 패턴, 롤링 설정도 다시 그대로 동작한다.

주의할 점이 두 가지 있다.

- `base.xml`은 FILE appender까지 항상 켠다. `logging.file.name`과 `logging.file.path`를 둘 다 쓰지 않으면 임시 폴더의 `spring.log`에 파일을 쓴다. 콘솔만 필요하면 필요한 부품만 골라서 include한다.

```xml
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
    <include resource="org/springframework/boot/logging/logback/console-appender.xml"/>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
    </root>
</configuration>
```

- `file-appender.xml`만 따로 include할 때는 `logging.file.name`이 꼭 있어야 한다. 값이 없으면 변수 자리에 `LOG_FILE_IS_UNDEFINED`가 들어가서 그 이름의 파일이 생긴다. `base.xml`은 이 문제를 막으려고 `LOG_FILE`의 기본값을 먼저 정해둔다.

## Spring 전용 태그

`logback-spring.xml`에서만 쓸 수 있는 태그이다.

### springProfile: 프로필마다 다른 설정

```xml
<springProfile name="local">
    <logger name="com.shrona.mommytalk" level="DEBUG"/>
</springProfile>

<springProfile name="prod-admin | prod-webapp">
    <root level="INFO">
        <appender-ref ref="FILE"/>
    </root>
</springProfile>
```

local에서는 mommytalk 패키지를 DEBUG로 두고, 운영 프로필에서만 root에 FILE을 연결하는 예이다. `name`에는 프로필 이름 하나 또는 표현식을 쓴다. `|`는 또는, `&`는 그리고, `!`는 아님이다. 예를 들어 `!local`은 local이 아닌 모든 프로필이다.

### springProperty: YAML 값을 XML에서 쓰기

```xml
<springProperty scope="context" name="appName" source="spring.application.name" defaultValue="app"/>

<!-- appender 안에서 -->
<file>logs/${appName}.log</file>
```

`source`에 적은 YAML 설정값을 `name`에 적은 이름의 변수로 가져온다. 값이 없으면 `defaultValue`를 쓴다.

## XML이 필요한 경우

값만 바꾸는 것은 YAML로 충분하다. 구조를 바꿔야 할 때 XML이 필요하다.

### ERROR만 모으는 파일 따로 두기

기본 설정에는 파일 appender가 하나뿐이라서 파일을 하나 더 두려면 XML이 필요하다. 필터로 ERROR 이상만 통과시킨다.

```xml
<appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        <level>ERROR</level>
    </filter>
    <file>logs/error.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <fileNamePattern>logs/error.%d{yyyy-MM-dd}.log.gz</fileNamePattern>
        <maxHistory>30</maxHistory>
    </rollingPolicy>
    <encoder>
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
</appender>

<root level="INFO">
    <appender-ref ref="CONSOLE"/>
    <appender-ref ref="ERROR_FILE"/>
</root>
```

- `ThresholdFilter`: 지정한 레벨 이상만 통과시킨다.
- `LevelFilter`: 지정한 레벨 하나만 골라낸다. WARN만 따로 모으는 것처럼 정확히 한 레벨이 필요할 때 쓴다.

### 특정 패키지 로그만 별도 파일로

앞의 additivity 예처럼 `<logger>`에 전용 appender를 붙이고 `additivity="false"`를 준다. 외부 API 호출 로그나 스케줄러 로그처럼 따로 보고 싶은 로그를 분리할 때 쓴다.

### 비동기로 기록하기 (AsyncAppender)

파일에 쓰는 동안 요청을 처리하는 스레드는 디스크 쓰기가 끝날 때까지 기다린다. `AsyncAppender`는 로그를 큐에 넣어두고 별도 스레드가 대신 쓰게 한다.

```xml
<!-- FILE appender는 따로 정의되어 있다고 가정 -->
<appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="FILE"/>
</appender>

<root level="INFO">
    <appender-ref ref="ASYNC_FILE"/>
</root>
```

대신 로그를 버릴 수 있다는 점을 알고 써야 한다. Logback 1.5.18 코드로 확인한 기본 동작은 다음과 같다.

- 큐 크기는 256이다.
- 큐의 남은 자리가 20% 아래로 떨어지면(80% 넘게 차면) TRACE, DEBUG, INFO 로그를 버린다. WARN과 ERROR는 남긴다.
- 모든 로그를 남기려면 `<discardingThreshold>0</discardingThreshold>`를 준다. 이 경우 큐가 가득 차면 요청 스레드가 자리가 날 때까지 기다린다.

### XML 없이 YAML로 되는 것

XML이 필요할 것 같지만 YAML로 되는 것도 있다. 맨 처음에 본 기본 설정 파일에 변수 자리가 마련된 것들이다.

- 콘솔과 파일에 다른 레벨 적용: `logging.threshold.console`, `logging.threshold.file` (기본 appender의 `ThresholdFilter` 빈칸)
- JSON 출력: `logging.structured.format.console`, `logging.structured.format.file` (5단계에서 다룬다)

## 직접 확인해보기

기본 설정 파일은 Gradle 캐시에 있는 Spring Boot jar에서 바로 꺼내볼 수 있다.

```bash
SB=$(find ~/.gradle/caches -name "spring-boot-3.4.5.jar" | head -1)
unzip -p "$SB" org/springframework/boot/logging/logback/file-appender.xml
unzip -p "$SB" org/springframework/boot/logging/logback/base.xml
```

아직 해보지 않은 실습으로, local에서 아래 순서로 해보면 "XML을 만들면 YAML 설정은 어떻게 되나"를 직접 확인할 수 있다.

1. YAML에 `logging.file.name`을 둔 채로, 위의 가장 작은 설정(CONSOLE만 있는 XML)으로 `logback-spring.xml`을 만들고 실행한다. 예상: 로그 파일이 생기지 않는다.
2. XML 내용을 `base.xml` include 한 줄로 바꾸고 다시 실행한다. 예상: YAML에 적은 경로에 로그 파일이 다시 생긴다.

## 정리

- YAML의 `logging.*`은 Spring Boot 기본 Logback 설정의 빈칸을 채우는 값이고, `logback-spring.xml`은 설정 자체를 직접 쓰는 것이다.
- `<springProfile>`, `<springProperty>`를 쓰려면 `logback.xml`이 아니라 `logback-spring.xml`이어야 한다.
- appender는 `<appender-ref>`로 연결해야 동작하고, 로그는 부모 logger의 appender로도 전달된다(additivity).
- 롤링 주기는 파일 이름 패턴의 `%d{...}`가 정하고, `maxHistory`는 그 주기의 개수이다. `totalSizeCap`은 `maxHistory`가 있어야 동작한다.
- XML을 만들어도 `logging.level`은 계속 적용되지만, 파일, 패턴, 롤링 설정은 XML이 변수를 참조하거나 기본 설정을 include해야 적용된다.
- 값만 바꾸면 YAML, appender 추가, 로그 분리, 필터, 비동기처럼 구조를 바꾸면 XML을 쓴다.
