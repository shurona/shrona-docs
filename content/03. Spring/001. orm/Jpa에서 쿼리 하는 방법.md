---
tags:
  - jpa/query
  - jpa/method
  - entityGraph
  - fetch-join
---

# 기본으로 주어진 정보
User와 Friend의 관계가 있다고 가정한다.
```Java
@Entity
public class Friend {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;
    
    @ManyToOne
    @JoinColumn(name = "friend_id")
    private User friendUser;
    
    // getter, setter 등
}

```
# User를 기준으로 Friend 목록을 검색하는 방법
## 메소드 이름 규칙을 활용
```Java
public interface FriendRepository extends JpaRepository<Friend, Long> {
    // User 객체로 친구 목록 검색
    List<Friend> findByUser(User user);
    
    // userId로 친구 목록 검색
    List<Friend> findByUserId(Long userId);
}

```
## `@Query` 어노테이션 활용
더 복잡한 쿼리가 필요하다면 `@Query`어노테이션을 활용할 수 있다.`
```Java
public interface FriendRepository extends JpaRepository<Friend, Long> {
    @Query("SELECT f FROM Friend f WHERE f.user = :user")
    List<Friend> findFriendListByUser(@Param("user") User user);
    
    @Query("SELECT f FROM Friend f WHERE f.user.id = :userId")
    List<Friend> findFriendListByUserId(@Param("userId") Long userId);
}

```
## 패치 조인을 활용해서 사용할 수 있다.
`N+1` 문제를 방지하기 위해서 하위 엔티티의 데이터를 함께 갖고 오고 싶으면 페치조인을 활용할 수 있다.
```Java
public interface FriendRepository extends JpaRepository<Friend, Long> {
    @Query("SELECT f FROM Friend f JOIN FETCH f.friendUser WHERE f.user = :user")
    List<Friend> findFriendListByUserWithFetchJoin(@Param("user") User user);
}
```
# User 안에 있는 Detail을 기준으로 조회하는 방법
## 메소드 이름 만으로 구현하기
언더스코어`(_)`를 사용해서 연관 관계의 필드에 접근할 수 있다.
```Java
public interface FriendRepository extends JpaRepository<Friend, Long> {
    // 기본 조회
    List<Friend> findByUser_Detail(String detail);
    
    // 부분 일치 검색
    List<Friend> findByUser_DetailContaining(String detailKeyword);
    
    // 시작/끝 패턴 검색
    List<Friend> findByUser_DetailStartsWith(String prefix);
    List<Friend> findByUser_DetailEndsWith(String suffix);
    
    // 정렬 추가
    List<Friend> findByUser_DetailOrderByCreatedDateDesc(String detail);
    
    // 복합 조건
    List<Friend> findByUser_DetailAndUser_Name(String detail, String name);
}
```
## EntityGraph와 함께 사용하기
`N+1` 문제를 방지하기 위해서 EntityGraph와 함께 사용할 수 있다.
```Java
@EntityGraph(attributePaths = {"user", "friendUser"})
List<Friend> findByUser_Detail(String detail);
```
## `@Query` 어노테이션 사용하기
JPQL을 직접 작성하는 방법   
다만 매개변수의 이름이 메소드 파라미터 이름과 동일하다면 `@Param` 어노테이션을 생략할 수 있다.
```Java
@Query("SELECT f FROM Friend f WHERE f.user.detail = :detail")
List<Friend> findByUserDetail(@Param("detail") String detail);

// 페치 조인 활용
@Query("SELECT f FROM Friend f JOIN FETCH f.user JOIN FETCH f.friendUser WHERE f.user.detail = :detail")
List<Friend> findFriendWithUserDetailAndFetch(@Param("detail") String detail);
```
# `@EntityGraph`와 fetch join의 차이
### EntityGraph
### 개념
- 기본 쿼리에 어떤 연관을 직접 로딩할지만 덧붙이는 선언적 방식
- JPA 구현체가 내부적으로 LEFT OUTER JOIN 을 삽입해서 즉시 로딩
### 장점
- 엔티티의 연관 관계 이름만 나열하면 되므로, 쿼리 문자열을 복잡하게 수정할 필요가 없다.
- 동일한 로딩 그래프를 여러 메서드에 재사용할 수 있도록 하나의 이름 기반 NamedEntityGraph를 엔티티에 선언해둘 수 있다.
### 단점
- 복잡한 조건부 조인(WHERE 절 연관)이나 필터링, 정렬 등 JPQL 로직과 섞어 쓰기는 어렵다.
## FETCH JOIN
### 개념
- 쿼리문 안에서 직접 조인 및 조건을 명시해서 연관 엔티티를 즉시 로딩하는 명시적 방식
- 개발자가 작성한 JOIN FETCH 구문을 그대로 반영
### 장점
- 조인과 함께 필터, 정렬, 그룹핑 등 JPQL의 모든 기능을 자유롭게 사용할 수 있다.
- 동적인 쿼리 작성이 필요한 경우(예: 특정 조건에서만 join), Criteria나 QueryDSL 없이도 바로 표현할 수 있다.
### 단점
- 쿼리 문자열이 길고 복잡해지기 쉽고, 여러 연관을 FETCH JOIN 하면 중복된 루트 결과 처리를 위해 DISTINCT를 추가해야 할 수도 있습니다.
# 코드적 트러블 슈팅
## 지속적인 validation 에러
### 원인
아래와 같이 Method를 이용한 방식과 @query를 이용한 방식을 사용해서 쿼리를 두 가지 방식으로 작성하였다.
```Java
@Query("SELECT f FROM friend f WHERE f.user = :user AND f.request = :request")  
    List<Friend> findUserListByUserAndRequest(@Param("user") User user, @Param("request") FriendRequest request);  
  
  
List<Friend> findUserListByUserAndRequest(User user, FriendRequest request);
```
그런데 메소드 방식은 문제가 없었으나 @Query를 사용해서 구현을 할 경우 아래와 같은 에러가 지속적으로 발생하였다.
```Java
Reason: Validation failed for query for method public abstract
```
### 해결
**테이블의 알파벳 대소문자**를 지켜서 `@Query`를 작성해야 했다.
```Java
// 기존에는 대문자로 적었으나 소문자로 적었어야 했다.
Friend => friend
```