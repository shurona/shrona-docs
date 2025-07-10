---
tags:
---
# ElementCollection과 Embeddable의 차이

## 값 타입 컬렉션이란?
값 타입 컬렉션은 JPA에서 엔티티가 아닌 값 타입(예: `String`, `Integer`, 혹은 `@Embeddable`로 정의된 클래스)들을 컬렉션(List, Set 등) 형태로 관리하는 기능입니다. 한 엔티티가 여러 개의 값 타입 데이터를 가질 때 사용됩니다.

### 주요 특징
- **별도의 테이블에 저장**: 컬렉션의 각 값은 엔티티의 기본키와 연관되어 저장됩니다.
- **생명주기 의존**: 부모 엔티티 삭제 시 컬렉션 데이터도 함께 삭제됩니다.
- **활용 예시**:
```java
@ElementCollection
@CollectionTable(name = "holiday_type", joinColumns = @JoinColumn(name = "holiday_id"))
@Column(name = "type", length = 100")
private Set<String> types = new HashSet<>();
```

## @ElementCollection과 @Embeddable의 차이

| 구분               | @ElementCollection                | @Embeddable                  |
|--------------------|------------------------------------|------------------------------|
| **목적**          | 값 타입 컬렉션을 엔티티에 매핑    | 값 타입(임베디드 타입) 정의 |
| **용도**          | 여러 개의 값 타입(혹은 임베디드 타입) 객체를 컬렉션으로 관리 | 여러 필드를 하나의 값 타입으로 묶어 표현 |
| **테이블 구조**    | 별도의 컬렉션 테이블 생성         | 엔티티 테이블에 필드가 합쳐져 저장 |
| **생명주기**       | 부모 엔티티에 종속                | 부모 엔티티에 종속          |
| **예시**           | `List<Address>`와 같이 여러 개 관리 | `Address` 한 개만 관리      |

### 예제 코드
#### 단일 값 타입
```java
@Embeddable
public class Address {
    private String city;
    private String street;
}

@Entity
public class Member {
    @Embedded
    private Address address;
}
```

#### 컬렉션 값 타입
```java
@Embeddable
public class Address {
    private String city;
    private String street;
}

@Entity
public class Member {
    @ElementCollection
    private List<Address> addressHistory = new ArrayList<>();
}
```

## 요약
- `@Embeddable`은 값 타입을 정의하는 용도, `@ElementCollection`은 그 값 타입을 컬렉션으로 엔티티에 매핑할 때 사용합니다.
- 값 타입 컬렉션은 별도의 테이블에 저장되며, 부모 엔티티와 생명주기를 함께 합니다.
- 단일 값 타입은 `@Embedded`, 컬렉션 값 타입은 `@ElementCollection`을 사용합니다.