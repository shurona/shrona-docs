---
tags:
  - Converter
  - cast
---

# 개념
- `@Converter`는 Java Spring, 특히 JPA(Java Persistence API) 환경에서 특정 데이터 타입을 다른 타입으로 변환할 때 사용하는 어노테이션이다. 
- 주로 엔티티(Entity) 클래스의 필드와 데이터베이스 컬럼 간의 타입이 다를 때 활용된다.
# 역할 및 동작방식
- 어노테이션을 사용해서 Converter 클래스 임을 지정한다.
	- @Converter 어노테이션을 붙인 클래스는 JPA가 관리하는 타입 변환 클래스로 인식된다.
	- `javax.persistence.AttributeConverter<X, Y>` 인터페이스를 구현해야 한다.
		- X는 엔티티 Field 타입 , Y는 DB Column 타입이다.
- 자동 변환하는 기능을 제공해 준다.
	- @Converter를 이용해 변환 로직을 작성하면, JPA가 엔티티를 저장하거나 조회할 때 자동으로 변환해준다.
- 적용 방법
	- 작성한 클래스에는 Conveter 어노테이션을 붙여서 Converter를 선언한다.
	- 사용할 엔티티 필드에는 `@Convert(converter = PhoneConverter.class)`를 붙여 어떤 변환기를 사용할지 지정한다
# 예시 코드
- 아래는 Entity는 휴대전화 형식으로 DB에서는 String 형식으로 저장되어 있을 때 자동변환하는 코드예시 
```Java
// autoApply = true를 사용하면 모든 PhoneNumber 필드에 자동 적용  
@Converter(autoApply = true)
public class PhoneNumberConverter 
	implements AttributeConverter<PhoneNumber, String> {  
  
    @Override  
    public String convertToDatabaseColumn(PhoneNumber phoneNumber) {  
        return (phoneNumber != null) ? phoneNumber.getPhoneNumber() : null;  
    }  
  
    @Override  
    public PhoneNumber convertToEntityAttribute(String dbData) {  
        return (dbData != null) ? new PhoneNumber(dbData) : null;  
    }  
}
```
- 엔티티에서 사용 코드
```Java
@Convert(converter = PhoneNumberConverter.class)
private PhoneNumber isActive;
```
# Cast 사용해서 변환
## 사용 SQL
- PhoneNumber 형식인 Entity를 String으로 비교하기 위해서 CAST()를 사용해서 변환한다.
```sql
SELECT clu  
FROM ChannelLineUser clu  
JOIN clu.lineUser lu  
WHERE clu.channel = :channel  
AND CAST(lu.phoneNumber AS string) LIKE %:query%
```
## Converter와 비교
| **목적**       | CAST() **(SQL/JPQL)**       | @Converter **(JPA** AttributeConverter**)**  |
| ------------ | --------------------------- | -------------------------------------------- |
| **타입 변환 시점** | _쿼리 실행 시_ DB 엔진이 처리         | _엔티티 ↔ DB_ 직·역직렬화 단계에서 JPA가 호출               |
| **주요 용도**    | 검색·JOIN 조건 안에서 **일시적** 형 변환 | 영속 계층 전반에 걸친 **항상** 동일 규칙의 변환                |
| **성능 영향**    | 인덱스 미적용 => 풀스캔 필요           | 변환 자체 비용은 미미<br>조회 건수가 많으면 객체 생성·GC 부하 발생 가능 |
