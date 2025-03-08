---
tags:
  - jpa
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