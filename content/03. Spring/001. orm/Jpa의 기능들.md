---
tags:
  - jpa
  - audit
  - embedded
  - vo
---
# 영속성 전이
영속성 전이는 JPA에서 특정 엔티티를 영속 상태로 만들 때 연관된 엔티티도 함께 영속성 상태로 만들고 싶을 때 사용한다.
영속 상태의 Entity에서 수행되는 작업들이 연관된 Entity 까지 전파되는 상황을 의미.
**Cascade / orphan**
### Cascade 종류
- ALL
- PERSIST
	- 부모 엔티티가 저장될 때 자식 엔티티도 함께 저장
- REMOVE
	- 부모 엔티티가 삭제될 때 자식 엔티티도 함께 삭제
- MERGE
	- 부모 엔티티가 merge될 때 자식 엔티티도 함께 merge
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

### SpringSecurity 에서 Audit User 찾기
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

### Request에서 Audit User 찾기
```Java
public class RequestUserJpaAuditConfig implements AuditorAware<String> {  
  
    @Override  
    public Optional<String> getCurrentAuditor() {  
  
        ServletRequestAttributes attributes  
            = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();  
  
        String userId = null;  
        if (attributes != null) {  
            HttpServletRequest request = attributes.getRequest();  
            String userIdHeader = request.getHeader(X_USER_ID);  
            if (userIdHeader != null) {  
                userId = userIdHeader;  
            }  
        }  
  
        return Optional.ofNullable(userId);  
    }  
}
```
1. **요청 컨텍스트 조회**  
    `RequestContextHolder`에서 현재 요청 정보(`ServletRequestAttributes`)를 가져옵니다.  
    → **주의**: 웹 컨텍스트가 아닌 경우(예: 배치 작업) `attributes`가 `null`일 수 있습니다.
2. **HTTP 요청 객체 추출**  
    웹 요청이 존재하는 경우(`attributes != null`), `HttpServletRequest`를 가져옵니다.
3. **헤더에서 사용자 ID 추출**  
    `X_USER_ID` 헤더 값을 읽어 `userId` 변수에 저장합니다.  
    → 헤더가 없는 경우 `userId`는 `null`이 됩니다.
4. **결과 반환**  
    `userId`를 `Optional`로 감싸서 반환한다.
    → `userId`가 `null`이면 `Optional.empty()`가 반환된다.
#### 문제 가능성
Batch Job과 같이 웹 요청이 아닌 경우에는 null일 수 있다.   
null인 경우에는 기본값이나 Annonymous와 같이 처리가 가능할 수 있다.

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

# Jpa를 사용해서 row 삭제
## 삭제 방법
### 메소드 사용
```Java
// deleteById 메소드 예시 
repository.deleteById(1L);

// deleteAllByIdIn 메소드 예시 List<Long> ids = Arrays.asList(1L, 2L, 3L);  
repository.deleteAllByIdIn(ids);  
```
### @Query 어노테이션 사용
```Java
// @Query 어노테이션을 활용한 삭제 예시  
@Query("delete from Member m where m.name = ?1") @Modifying void deleteByName(String name);
```
## 발생한 문제
- 연관된 자식 엔티티가 삭제되지 않아서 삭제 쿼리가 날라가지 않는 현상 발생
### 예제
- 아래와 같이 User와 Board Entity가 있다고 가정하면
```Java
@Entity
public class User {
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Board> boards = new ArrayList<>();
}
```

```Java
@Entity
public class Board {
    @ManyToOne(fetch = FetchType.LAZY)
    private User user;
}
```
### 원인
- 연관된 Board들이 User의 boards 리스트에 그대로 남아 있기 때문이다.
- JPA는 Board들이 `고아(orphan)`가 아니라고 판단해서 삭제하지 않음.
### 해결
- 아래와 같이 삭제를 진행하면 된다.
```Java
user.getBoards().clear(); // 양방향 연관관계를 끊음 → orphanRemoval 작동
userRepository.delete(ids);
```
