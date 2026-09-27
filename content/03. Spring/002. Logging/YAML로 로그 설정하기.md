---
tags:
  - logging
  - spring-boot
  - yaml
---
# YAML로 로그 설정하기

> [[배포 후 로그 관리를 해보자]] 로드맵의 2단계

[[SLF4J와 Logback의 관계]]에서 본 Level, Appender, Encoder를 YAML 설정으로 직접 바꿔본 기록이다.
`logback-spring.xml`을 만들지 않고 현재 application YAML에 설정을 추가했다.

## 프로필마다 로그 레벨 다르게 두기

공통 설정에서는 root Logger를 INFO로 명시했다.

```yaml
# application.yaml
logging:
  level:
    root: INFO
```

root를 DEBUG로 바꾸면 Spring, Hibernate 같은 라이브러리 로그까지 한꺼번에 늘어난다.
로컬에서 자세히 보고 싶은 것은 mommytalk 코드이므로 `application-local.yaml`에서는 애플리케이션 패키지만 DEBUG로 설정했다.

```yaml
# application-local.yaml
logging:
  level:
    com.shrona.mommytalk: DEBUG
```

운영 프로필에서는 같은 패키지를 INFO로 두었다.

```yaml
# application-prod-admin.yaml, application-prod-webapp.yaml
logging:
  level:
    com.shrona.mommytalk: INFO
```

local 프로필로 실행하고 기존 `log.debug` 코드가 실행되는 요청을 보냈더니 DEBUG 로그가 콘솔에 출력됐다. 1단계에서 본 Logger의 이름 계층과 레벨 상속이 실제 설정에서도 그대로 동작한 것이다.

여기서 `debug: true`는 사용하지 않았다. 이 옵션은 모든 Logger를 DEBUG로 바꾸는 설정이 아니라 Spring Boot가 정해둔 일부 Logger의 상세 로그를 켜는 옵션이다. mommytalk 코드의 DEBUG 로그를 보려면 위처럼 패키지 Logger의 레벨을 직접 지정해야 한다.

## 콘솔 로그를 파일에도 남기기

같은 local 설정에 `logging.file.name`을 추가했다.

```yaml
logging:
  level:
    com.shrona.mommytalk: DEBUG
  file:
    name: mommy-logging/mommytalk.log
```

서버를 다시 실행하자 프로젝트 루트의 `mommy-logging/mommytalk.log`가 생성됐다. 콘솔 출력은 그대로 유지됐고, 콘솔에서 확인한 DEBUG 로그도 파일에 함께 기록됐다. `logging.file.name`을 지정하면 Spring Boot 기본 Logback 설정에서 기존 콘솔 출력과 함께 파일 출력도 활성화된다.

`logging.file.name` 값이 기본 설정의 어디에 들어가는지는 [[logback-spring.xml 이해하기]]에서 다룬다.

상대 경로는 애플리케이션의 현재 작업 디렉토리를 기준으로 잡힌다. 이번 실습에서는 로그라는 용도가 드러나도록 `mommy-logging/` 디렉토리를 사용했고, 생성된 로그 파일이 Git에 포함되지 않도록 디렉토리 전체를 `.gitignore`에 추가했다.

## 콘솔과 파일의 출력 형식 나누기

콘솔과 파일의 차이가 보이도록 콘솔에는 시각만, 파일에는 날짜와 시각을 넣었다.

```yaml
logging:
  pattern:
    console: "%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
```

서버를 다시 실행하고 같은 요청을 보내니 콘솔과 파일에 서로 다른 형식으로 로그가 기록됐다. 1단계에서 본 Encoder와 Pattern을 `logging.pattern.console`, `logging.pattern.file`로 각각 설정할 수 있다는 것을 확인했다.

## 파일 크기로 롤링하기

롤링을 바로 확인할 수 있도록 파일 크기 제한을 10KB로 낮췄다.

```yaml
logging:
  logback:
    rollingpolicy:
      max-file-size: 10KB
```

기존 `mommytalk.log`가 10KB를 넘은 상태에서 서버를 다시 실행하고 로그를 발생시키자 `mommytalk.log.2026-08-08.0.gz`가 생겼다. 이후 로그는 새로 만들어진 `mommytalk.log`에 계속 기록됐다.

롤링이 일어날 때는 그 시점의 `mommytalk.log` 전체가 하나의 압축 파일이 된다. 같은 날 다시 크기 제한을 넘으면 `.1.gz`, `.2.gz`처럼 번호가 붙은 파일이 하나씩 늘어난다. 기존 압축 파일들을 다시 합쳐서 압축하는 방식은 아니다.

10KB는 동작을 빠르게 보기 위한 값이다. Spring Boot 기본값은 10MB이므로 실습이 끝나면 제한을 원래대로 돌리거나 실제 운영 기준에 맞게 다시 정해야 한다.

## 보관 기간과 전체 용량 제한하기

크기 기준 롤링을 확인한 뒤 실습용 10KB를 10MB로 돌리고 보관 정책을 추가했다.

```yaml
logging:
  logback:
    rollingpolicy:
      max-file-size: 10MB
      max-history: 7
      total-size-cap: 100MB
```

현재 파일 이름 패턴에는 날짜가 들어가므로 `max-history: 7`은 최근 7일 구간의 압축 파일을 보관한다는 뜻이다. 같은 날 크기 때문에 여러 파일로 나뉘어도 `.0.gz`, `.1.gz` 파일은 같은 날짜 구간에 속한다.

`total-size-cap: 100MB`는 압축 파일 전체 크기의 상한이다. 먼저 보관 기간을 지난 파일을 지우고, 남은 파일이 100MB를 넘으면 오래된 파일부터 더 지운다. 정리는 보통 다음 롤링이 발생할 때 실행되므로 설정 직후 파일이 바로 지워지는 것은 아니다.

설정을 적용한 뒤 서버가 오류 없이 실행되는 것을 확인했다. 날짜, 크기, 보관 기간을 제어하는 것까지는 YAML로 충분하다. YAML로 부족한 경우는 [[logback-spring.xml 이해하기]]에서 정리했다.

실습에 쓴 local의 파일 출력 설정은 이후 제거했다. 운영에서 로그 파일을 어디에 남길지는 Docker 볼륨 마운트와 함께 정해야 해서 4단계에서 다룬다.

## 정리

- YAML의 `logging.level`로 프로필별 로그 레벨을 나눌 수 있다. local에서는 mommytalk 패키지만 DEBUG로 두고 운영에서는 INFO로 유지했다.
- `logging.file.name`을 설정하면 기존 콘솔 출력에 파일 출력이 추가된다.
- `logging.pattern.console`과 `logging.pattern.file`로 두 출력의 형식을 따로 지정할 수 있다.
- `logging.logback.rollingpolicy.max-file-size`로 파일 크기 기준 롤링을 설정할 수 있다. 롤링된 파일은 날짜와 번호가 붙은 `.gz`로 압축된다.
- `max-history`와 `total-size-cap`으로 압축 파일의 보관 기간과 전체 용량을 제한할 수 있다.
- YAML로 부족할 때 쓰는 `logback-spring.xml`은 [[logback-spring.xml 이해하기]]에서 다룬다.
