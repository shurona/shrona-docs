---
tags:
  - LocalDateTime
  - Instant
  - ZonedDateTime
---
# LocalDateTime
## 특징
- 날짜와 시간 정보만 있음(예: 2025-06-10T08:30:00)
  (yyyy-MM-dd hh:mm:ss)
- 타임존(시간대) 정보 없음
- 서버의 기본 타임존(JVM 타임존)에 의존
- 주로 로컬 환경에서만 유효, 글로벌 서비스에서는 혼동 우려
## 사용 예시
```Java
LocalDateTime now = LocalDateTime.now();
```
# ZonedDateTime
## 특징
- 날짜, 시간, 타임존(ZoneId) 정보를 모두 가짐
- ex> 2025-06-10T08:30:00+09:00Asia/Seoul
- 시간대가 명확히 포함되어 있어 글로벌 서비스에 적합하다
## 사용 예시
```Java
ZonedDateTime now = ZonedDateTime.now(ZoneId.of("Asia/Seoul"));
```
# OffsetDateTime
## 특징
- 날짜, 시간, UTC 기준의 오프셋(Offset, +09:00 등) 정보 포함
- ex> 2025-06-10T08:30:00+09:00
- 타임존 이름(Asia/Seoul 등)은 없고, 오프셋만 있음
- 시간대 이름이 필요 없고, 단순히 오프셋만 관리할 때 유용
## 사용 예시
```Java
OffsetDateTime now = OffsetDateTime.now(ZoneOffset.of("+09:00"));
```
# Instant
## 특징
- UTC 기준의 타임 스탬프
- 시간대 개념이 아예 존재 하지 않는다 (UTC)
- ex> 2025-06-09T23:30:00Z
- 주로 DB 저장, 네트워크 전송 등에서 표준 시각 기준으로 사용
## 사용 예시
```Java
Instant now = Instant.now();
```
# 타입 간 변환 예시
```Java
// LocalDateTime → ZonedDateTime
LocalDateTime ldt = LocalDateTime.now();
ZonedDateTime zdt = ldt.atZone(ZoneId.of("Asia/Seoul"));

// ZonedDateTime → OffsetDateTime
OffsetDateTime odt = zdt.toOffsetDateTime();

// ZonedDateTime → Instant
Instant instant = zdt.toInstant();

// Instant → ZonedDateTime
ZonedDateTime zdt = instant.atZone(ZoneId.of("Asia/Seoul"));

```
# Spring에서 Default UTC 설정
## JVM 옵션으로 설정
```sh
java -Duser.timezone=UTC -jar myapp.jar
```
## 코드에서 설정
- 애플리케이션 시작 시 타임존을 UTC로 강제한다.
```Java
@PostConstruct
public void setTimeZone() {
    TimeZone.setDefault(TimeZone.getTimeZone("UTC"));
}
```
## applicatoin.yaml에서 설정하기
- jackson 및 jpa에 적용이 가능하다.
```yaml
spring:
  jackson:
    time-zone: UTC
  jpa:
    properties:
      hibernate:
        jdbc:
          time_zone: UTC
```
