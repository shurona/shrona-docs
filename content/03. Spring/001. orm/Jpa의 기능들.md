---
tags:
  - jpa
  - audit
---
# 영속성 전이
영속성 전이는 JPA에서 특정 엔티티를 영속 상태로 만들 때 연관된 엔티티도 함께 영속성 상태로 만들고 싶을 때 사용한다.
영속 상태의 Entity에서 수행되는 작업들이 연관된 Entity 까지 전파되는 상황을 의미.
**Cascade / orphan**
### Cascade 종류
- ALL
- PERSIST
- REMOVE
- MERGE
- REFRESH
- DETACH

### 주의사항
완전히 종속일 때만 사용하는 것이 좋다. 만약 child를 다른 곳에서 알게 되면 사용하지 않는 것이 좋다.    
⇒ 단일 엔티티에 완전히 종속적일 때 사용한다.
- 라이프 사이클이 거의 유사할 때
- 단일 소유자일 때

orphanRemove는 ManyToOne에는 존재하지 않는다. ⇒ 삭제되는 entity가 다른 곳에서 참조될 가능성이 높기 때문이다.

# JPA Auditing
[spring audit docs](https://docs.spring.io/spring-data/jpa/reference/auditing.html)
생성일자, 수정일자, 식별자와 같은 관계형 데이터베이스에서 테이블에 매핑할 때 도메인들이 공통적으로 갖고 있는 필드 또는 칼럼들을 관리해주는 기능을 말한다.
Spring Data Jpa에서 시간에 대해서 자동으로 값을 넣어주는 기능   
아래의 두 어노테이션을 설정해서 사용할 수 있다.   
- `@EnableJpaAuditing`
- `@EntityListners(AuditingEntityListener.class)`
## AuditorAware
`@CreatedBy` 또는 `@LastModifiedBy` 를 사용하기 위해서는 audit infrastructure에서 현재 principal을 인식해야할 필요가 있다. ⇒ 그것을 위해 사용하는 것이 `AuditAware<T>`
```Java
class SpringSecurityAuditorAware implements AuditorAware<User> {

  @Override
  public Optional<User> getCurrentAuditor() {

    return Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .filter(Authentication::isAuthenticated)
            .map(Authentication::getPrincipal)
            .map(User.class::cast);
  }
}
```

Spring Security에서 제공하는 Authentication object에서 접근해서 UserDetails 인스턴스를 조회해서 알게 된다.
## Audit을 기록하기 위한 상위 클래스를 만드는 방법
### 예제
```Java
@Getter  
@MappedSuperclass  
@EntityListeners(AuditingEntityListener.class)  
public class BaseEntity {  
    @Column(name = "created_at", updatable = false)  
    @CreatedDate  
    protected LocalDateTime createdAt;  
  
    @CreatedBy  
    @Column(updatable = false)  
    protected String createdBy;  
  
    @Column(name = "updated_at")  
    @LastModifiedDate  
    protected LocalDateTime updatedAt;  
  
    @LastModifiedBy  
    protected String updatedBy;  
  
}
```
#### MappedSuperclass
- 이 클래스는 JPA의 **매핑된 슈퍼클래스**로 정의됩니다.
- 즉, 이 클래스를 상속받는 엔티티 클래스들은 이 클래스의 필드들을 자신의 테이블에 포함하게 됩니다.
- 하지만 `BaseEntity` 자체는 테이블로 매핑되지 않습니다.
#### EntityListeners()
- Spring Data JPA의 **Audit 기능**을 활성화하기 위해 사용됩니다.
- `AuditingEntityListener`는 엔티티가 저장되거나 업데이트될 때, 자동으로 생성자와 수정자 정보를 기록합니다.
- 이를 통해 `@CreatedDate`, `@CreatedBy`, `@LastModifiedDate`, `@LastModifiedBy`와 같은 어노테이션이 동작하게 됩니다.
# `Embedded`와 `Embeddable`이란
## Embeddable 정의
`@Embeddable`은 값 타입(VO, Value Object) 클래스를 정의할 때 사용하는 어노테이션이다.   
이 어노테이션이 붙은 클래스는 엔티티의 일부(속성)로 저장되며, 자체적으로 테이블을 만들지 않고, 엔티티의 테이블에 컬럼으로 포함이 된다.
### 예시
```Java
@Embeddable
public class Address {
    private String city;
    private String street;
    private String zipcode;
    // 생성자, getter 등
}
```
## Embedded 정의
`@Embedded`는 **엔티티의 필드**에 붙여, 해당 필드가 `@Embeddable`로 정의된 값 타입임을 명시한다.   
이 필드는 엔티티의 테이블에 컬럼으로 포함되어 저장됩니다
### 예시
Address의 필드(city, street, zipcode)가 User 테이블에 컬럼으로 표시되어서 저장된다.
```java
@Entity
public class User {
    @Id
    private Long id;
    private String name;

    @Embedded
    private Address address; 
}

```
### 특징
- 불변 객체 권장 
	- 값 타입은 불변(immutable)으로 설계하는 것이 좋으며, setter를 두지 않는 것이 일반적입니다
- 임베디드 값의 내부 필드 변경만으로는 JPA가 변경을 감지하지 못할 수 있다. 
	- 값 객체는 불변으로 만들고 새 객체로 교체하는 방식이 권장됩니다
>  ### @NoArgsConstructor(access = AccessLevel.PROTECTED)
>	- JPA Entity 요구사항: JPA에서 엔티티 클래스를 사용할 때 기본 생성자가 반드시 필요하다
>	- 다만 무분별한 객체의 생성을 방지하기 위해서 `PROTECTED`로 선언해준다.