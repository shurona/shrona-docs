# 데이터 접근 기술 - MyBatis
## 대표 설정 값
- `mybatis.type-aliases-package`
	- 마이바티스에서 타입 정보를 사용할 때는 패키지 이름을 적어주어야 하는데, 
	  여기에 명시하면 패키지 이름을 생략할 수 있다.
	- 지정한 패키지와 그 하위 패키지가 자동으로 인식된다.
	- 여러 위치를 지정하려면 `,` , `;`로 구분하면 된다.
- `mybatis.configuration.map-underscore-to-camel-case
	- JdbcTemplate의 `BeanPropertyRowMapper` 에서 처럼 언더바를 카멜로 자동 변경해주는 기능
## 적용 기본
### 자바 파일
```Java
@Mapper
public interface ItemMapper {
	void save(Item item);
	
	void update(@Param("id") Long id, 
				@Param("updateParam") ItemUpdateDto updateParam);
				
	Optional<Item> findById(Long id);
	
	List<Item> findAll(ItemSearchCond itemSearch);
}
```
- 파라미터가 2개면 `@Param`을 붙여줘야 한다.
- 이 인터페이스에는 `@Mapper` 애노테이션을 붙여주어야 한다. 그래야 MyBatis에서 인식할 수 있다.
- 이 인터페이스의 메서드를 호출하면 다음에 보이는 `xml` 의 해당 SQL을 실행하고 결과를 돌려준다.
- 구현체는 자동으로 만들어준다.
### xml 파일
-  자바 코드가 아니기 때문에 `src/main/resources` 하위에 만들되, 패키지 위치는 맞추어 주어야 한다
```xml

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
"http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="hello.itemservice.repository.mybatis.ItemMapper">
	<insert id="save" useGeneratedKeys="true" keyProperty="id">
		insert into item (item_name, price, quantity)
		values (#{itemName}, #{price}, #{quantity}) => get으로 바인딩 된다.
	</insert>
	<update id="update">
		update item
		set item_name=#{updateParam.itemName},
			price=#{updateParam.price},
			quantity=#{updateParam.quantity}
		where id = #{id}
	</update>
	<select id="findById" resultType="Item">
		select id, item_name, price, quantity
		from item
		where id = #{id}
	</select>
	<select id="findAll" resultType="Item">
		select id, item_name, price, quantity
		from item
		<where>
			<if test="itemName != null and itemName != ''">
				and item_name like concat('%',#{itemName},'%')
			</if>
			<if test="maxPrice != null">
				and price &lt;= #{maxPrice}
			</if>
		</where>
	</select>
</mapper>
```
- `useGeneratedKeys="true"` `keyProperty="id"`  => 키를 자동으로 생성해준다.
## 기본 쿼리문 알아보기
### INSERT
- insert SQL은 `<insert>` 를 사용한다.
- `id` 에는 매퍼 인터페이스에 설정한 메서드 이름을 지정하면 된다.
- 파라미터는 `#{}` 문법을 사용하면 된다. 그리고 매퍼에서 넘긴 객체의 프로퍼티 이름을 적어주면 된다
### UPDATE
- Update SQL은 `<update>` 를 사용하면 된다.
- 파라미터가 1개만 있으면 `@Param` 을 지정하지 않아도 되지만, 파라미터가 2개 이상이면 `@Param`으로 이름을 지정해서 파라미터를 구분해야 한다.
### SELECT
- Select SQL은 `<select>` 를 사용하면 된다
- `resultType` 은 반환 타입을 명시하면 된다.
- `<where>` , `<if>` 같은 동적 쿼리 문법을 통해 편리한 동적 쿼리를 지원한다.
- `<where>` 은 적절하게 `where` 문장을 만들어준다.
- xml 특수문자(태그가 겹칠 수 있기 때문이다)
```text
< : &lt;
> : &gt;
& : &amp;
```
## 주요 내용
### 적용 방식
- 애플리케이션 로딩 시점에 MyBatis 스프링 연동 모듈은 `@Mapper` 가 붙어있는 인터페이스를 조사
- 해당 인터페이스가 발견되면 동적 프록시 기술을 사용해서 `ItemMapper` 인터페이스의 구현체를 만든다
- 생성된 구현체를 스프링 빈으로 등록한다.
### 매퍼 구현체
-  MyBatis에서 발생한 예외를 스프링 예외 추상화인 `DataAccessException` 에 맞게 변환해서 반환해준다
- 마이바티스 스프링 연동 모듈이 만들어주는 `ItemMapper` 의 구현체 덕분에 인터페이스 만으로 편리하게 XML의 데이터를 찾아서 호출할 수 있다.
## 동적 쿼리
### IF
```xml
<select id="findActiveBlogWithTitleLike" resultType="Blog">
	SELECT * FROM BLOG
	WHERE state = ‘ACTIVE’
	<if test="title != null">
		AND title like #{title}
	</if>
</select>
```
- 내부의 문법은 OGNL을 사용한다
### choose, when, otherwise
- 자바의 switch와 유사한 구문으로 사용가능하다
```xml
<select id="findActiveBlogLike"resultType="Blog">
	SELECT * FROM BLOG WHERE state = ‘ACTIVE’
	<choose>
		<when test="title != null">
			AND title like #{title}
		</when>
		<when test="author != null and author.name != null">
			AND author_name like #{author.name}
		</when>
		<otherwise>
			AND featured = 1
		</otherwise>
	</choose>
</select>
```
### foreach
```xml
<select id="selectPostIn" resultType="domain.blog.Post">
	SELECT *
	FROM POST P
	<where>
		<foreach item="item" index="index" collection="list"
		open="ID in (" separator="," close=")" nullable="true">
			#{item}	
		</foreach>
	</where>
</select>
```
- 컬렉션을 반복 처리할 때 사용한다. `where in (1,2,3,4,5,6)` 와 같은 문장을 쉽게 완성할 수 있다.
- 파라미터를 List로 전달하면 된다.
## 기타기능
### 별칭 사용 하지 않고 사용하기
- resultMap 사용
```xml
<resultMap id="userResultMap" type="User">
	<id property="id" column="user_id" />
	<result property="username" column="user_name"/>
	<result property="password" column="hashed_password"/>
</resultMap>

<select id="selectUsers" resultMap="userResultMap">
	select user_id, user_name, hashed_password
	from some_table
	where id = #{id}
</select>
```
# 스프링 트랜잭션의 이해
## 트랜잭션 추상화
- 스프링은 트랜잭션을 사용하는 입장에서 추상화를 통해서 사용을 하게 된다
- 스프링은 트랜잭션을 추상화해서 제공할 뿐만 아니라, 실무에서 주로 사용하는 데이터 접근 기술에 대한 트랜잭션 매니저의 구현체도 제공한다.
- 적절한 트랜잭션 매니저를 선택해서 스프링 빈으로 등록해준다.
## 트랜잭션 사용 방식
### 선언적 트랜잭션 방식
- `@Transactional` 애노테이션 하나만 선언해서 매우 편리하게 트랜잭션을 적용
- `@Transactional` 을 통한 선언적 트랜잭션 관리 방식을 사용하게 되면 기본적으로 프록시 방식의 AOP가 적용
```Java

public void logic() {
//트랜잭션 시작
	TransactionStatus status = transactionManager.getTransaction(..);
	try {
	//실제 대상 호출
		target.logic();
		transactionManager.commit(status); //성공시 커밋
	} catch (Exception e) {
		transactionManager.rollback(status); 
		throw new IllegalStateException(e);
	}
}
```
### 프로그래밍 방식의 트랜잭션 관리
- 트랜잭션 매니저 또는 트랜잭션 템플릿 등을 사용해서 트랜잭션 관련 코드를 직접 작성하는 것
### 도입 후 과정
1. 트랜잭션은 커넥션에 `con.setAutocommit(false)`를 지정하면서 시작한다.
2. 같은 트랜잭션을 유지하려면 같은 데이터베이스 커넥션을 사용해야 한다.
3. 이것을 위해 스프링 내부에서는 트랜잭션 동기화 매니저가 사용된다
4. `JdbcTemplate` 을 포함한 대부분의 데이터 접근 기술들은 트랜잭션을 유지하기 위해 내부에서 트랜잭션 동기화 매니저를 통해 리소스(커넥션)를 동기화
## 스프링 컨테이너에 트랜잭션 프록시 등록
- `@Transactional` 어노테이션이 특정 클래스나 메소드에 하나라도 있으면 트랜잭션 AOP는 프록시를 만들어서 스프링 컨테이너에 등록한다.
  그래서 실제 `basicService`객체 대신에 프록시인 `basicService$$CGLIB`을 스프링 빈에 등록한다.
  그리고 프록시는 내부에 있는 실제 `basicService`를 참조하게 된다.
- 클라이언트인 `txBasicTest`는 스프링 컨테이너에 의존관계인 `BasicService`주입을 요청하게 되고 이 때 실체 객체 대신에 프록시가 빈으로 등록되어 있기 때문에 프록시로 주입되게 된다.
- 프록시는 `BasicService`를 상속해서 만들어지기 때문에 다형성을 활용할 수 있다.
  따라서 주입받을 때 프록시 객체인 `BasicService$$CGLIB`를 주입 받을 수 있다
### TransactionSynchronizationManager.isActualTransactionActive()
- 현재 쓰레드에 트랜잭션이 적용되어 있는지 확인할 수 있는 메소드이다.
### 트랜잭션의 우선순위
- 클래스에 적용하면 포함된 메소드에 다 적용이 된다.
## 트랜잭션 AOP 주의 사항 - 프록시 내부 호출
- @Transactional을 감싸주면 클래스를 호출하면 프록시가 호출이 된다.
- 대상 객체 내부에서 메소드를 호출이 발생하면 프록시를 거치지 않고 대상 객체를 직접 호출하는 문제가 발생한다.
- 내부 호출에서는 프록시를 거치지 않기 때문에 트랜잭션이 걸리지 않는다. 
- 트랜잭션을 위해서는 internal을 다른 클래스로 분리해야 한다.
```Java
@Test  
void externalCall() {  
    callService.external();  // 클래스를 호출하고
}

static class CallService {  
  
    public void external() {     // 여기서 내부 internal 메소드를 호출하면 
        printTxInfo();           // 내부 호출로 프록시를 거치지 않기 때문에
        internal();              // Transactional이 걸리지 않는다.
    }  
  
    @Transactional  
    public void internal() {  
        printTxInfo();  
    }  
  
    private void printTxInfo() {  
        boolean txActive =  
            TransactionSynchronizationManager.isActualTransactionActive();  
        log.info("tx active={}", txActive);  
  
    }    
}
```

### 주의
- public 메소드만 트랜잭션을 적용할 수 있다.
	- 보통 의도하지 않은 곳 까지 과도하게 걸리지 않도록 방지하기 위함이다
## 트랜잭션 AOP 주의사항 - 초기화 시점
- 초기화 코드(예: `@PostConstruct` )와 `@Transactional` 을 함께 사용하면 트랜잭션이 적용되지 않는다.
  왜냐하면 초기화 코드가 먼저 호출이 되고 그 다음에 트랜잭션 AOP가 적용되기 때문이다.
- 확실한 대안은 `ApplicationReadyEvent`을 사용하는 것이다.
```
 Initialized JPA EntityManagerFactory for persistence unit 'default'
hello.springtx.apply.InitTxTest$Hello : Hello init @PostConstruct tx active=false
hello.springtx.apply.InitTxTest       : Started InitTxTest in 0.768 seconds
o.s.t.i.TransactionInterceptor         : Getting transaction for [hello.springtx.apply.InitTxTest$Hello.init2] => 트랜잭션이 올라오는 것을 확인할 수 있다.
hello.springtx.apply.InitTxTest$Hello   : Hello init ApplicationReadyEvent tx active=true
```
## 트랜잭션 옵션
### value, transactionManager
- 트랜잭션을 사용하려면 먼저 스프링 빈에 등록된 어떤 트랜잭션 매니저를 사용할 지 알아야 한다.
- 이 값을 생략하면 기본으로 등록된 트랜잭션 매니저를 사용하게 된다.
### rollbackFor
이 옵션을 사용하면 기본 정책에 추가로 어떤 예외가 발생할 때 롤백할 지 지정할 수 있다.

```java
@Transactional(rollbackFor = Exception.class)
```
이렇게 지정하면 체크 예외인 `Exception`(하위 예외들도 대상에 포함된다) 이 발생해도 롤백하게 된다. 
### propagation
TODO: 뒤에 설명 링크 걸어놓자
### isolation
- 트랜잭션 격리 수준을 지정할 수 있다. 
- 기본 값은 데이터베이스에서 설정한 트랜잭션 격리 수준을 사용하는 `DEFAULT`이다. 
- 대부분 데이터베이스에서 설정한 기준을 따른다. 
- 애플리케이션 개발자가 트랜잭션 격리 수준을 직접 지정하는 경우는 드물다.
### timeout
- 트랜잭션 수행 시간에 대한 타임아웃을 초 단위로 지정한다. 
- 기본 값은 트랜잭션 시스템의 타임아웃을 사용한다. 
- 운영환경에 따라 동작하는 경우도 있고 그렇지 않은 경우도 있기 때문에 꼭 확인하고 사용해야 한다.
### readonly
- 트랜잭션은 기본적으로 읽기 쓰기가 모두 가능한 트랜잭션이 생성된다.
- `readOnly=true` 옵션을 사용하면 읽기 전용 트랜잭션이 생성된다. 이 경우 등록, 수정, 삭제가 안되고 읽기 기능만 작동한다.
- `readOnly` 옵션을 사용하면 읽기에서 다양한 성능 최적화가 발생할 수 있다
- 주의
	- 드라이버나 데이터베이스에 따라 정상 동작하지 않는 경우도 있다.
#### 상세 내용
- JdbcTemplate은 읽기 전용 트랜잭션 안에서 변경 기능을 실행하면 예외를 던진다.
- JPA(하이버네이트)는 읽기 전용 트랜잭션의 경우 커밋 시점에 플러시를 호출하지 않는다
- 읽기 전용 트랜잭션의 경우 읽기(슬레이기브) 데이터베이스의 커넥션을 획득해서 사용한다.
## 예외와 트랜잭션 커밋, 롤백 -  기본
### 스프링은 왜 체크 예외는 컷하고 언 체크는 롤백할까
- 체크 예외는 비즈니스 의미가 있을 때 사용을 하고 런타임 예외는 복구 불가능한 예외로 가정한다.
- `rollbackFor`라는 옵션을 사용하면 체크 예외도 롤백시킬 수 있다.
### 주문 서비스 시 예시
- 정상
	- 주문시 결제를 성공하면 주문 데이터를 저장하고 결제 상태를 `완료` 처리
- 시스템 예외
	- 주문시 내부에 복구 불가능한 예외가 발생하면 전체 데이터를 `롤백`
- 비즈니스 예외
	- 주문시 결제 잔고가 부족하면 주문 데이터를 저장하고, 결제 상태를 `대기`
	- 시스템 예외가 아니라 비즈니스 상황이 예외인 것이다.
# 스프링 트랜잭션 전파1 - 기본
트랜잭션이 진행중일때 추가로 트랜잭션을 수행하면 어떻게 처리할지 결정하는 것을 트랜잭션 전파라고 한다.
## 전파 기본
- 스프링은 이해를 돕기 위해 물리 트랜잭션과 논리 트랜잭션으로 개념을 나눈다
- 물리 트랜잭션은 우리가 이해하는 실제 데이터베이스에 적용되는 트랜잭션을 뜻한다. 실제 커넥션을 통해서 트랜잭션을 시작(`setAutoCommit(false))` 하고, 실제 커넥션을 통해서 커밋, 롤백하는 단위이다.
- 논리 트랜잭션은 트랜잭션 매니저를 통해 트랜잭션을 사용하는 단위이다.
  이런 논리 트랜잭션은 트랜잭션이 진행되는 중에 내부에 트랜잭션을 사용하는 경우에 나타난다. 
### 원칙
- 모든 논리 트랜잭셔닝 커밋 되어야 물리 트랜잭셔닝 커밋된다.
- 하나의 논리 트랜잭션이라도 롤백되면 물리 트랜잭션은 롤백된다.
## 전파 예제
- 트랜잭션이 시작되어  있는 지 상태를 확인하기 위한 클래스
``` Java
TransactionStatus outer 
     = txManager.getTransaction(new DefaultTransactionAttribute());
     
outer.isNewTransaction()
```
- 하나의 로직 트랜잭션이 끝나더라도 물리 트랜잭션이 끝나지 않으면 아무일도 하지 않는다.
- 내부 트랜잭션을 시작할 때 `Participating in existing transaction` 이라는 메시지를 확인할 수 있다.
- 데이터베이스에 실제 커밋하는 것은 물리 트랜잭션만 가능하다.
## 스프링 트랜잭션 전파 - 내부 롤백
### 자바 코드
```Java
log.info("외부 트랜잭션 시작");  
TransactionStatus outer = txManager.getTransaction(new  
    DefaultTransactionAttribute());  
log.info("내부 트랜잭션 시작");  
TransactionStatus inner = txManager.getTransaction(new  
    DefaultTransactionAttribute());  
log.info("내부 트랜잭션 롤백");  
txManager.rollback(inner);  
log.info("외부 트랜잭션 커밋");  
assertThatThrownBy(() -> txManager.commit(outer))  
    .isInstanceOf(UnexpectedRollbackException.class);
```
### 로그
```
외부 트랜잭션 시작

Creating new transaction with name [null]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT

Acquired Connection [HikariProxyConnection@438448733 wrapping conn0: url=jdbc:h2:mem user=SA] for JDBC transaction

Switching JDBC Connection [HikariProxyConnection@438448733 wrapping conn0: url=jdbc:h2:mem user=SA] to manual commit

내부 트랜잭션 시작

Participating in existing transaction

내부 트랜잭션 롤백

# 논리 트랜잭션은 롤백을 할 수 없으므로 기존 트랜잭션을 롤백 전용을 표시한다.
Participating transaction failed - marking existing transaction as rollback-only

Setting JDBC transaction [HikariProxyConnection@438448733 wrapping conn0: url=jdbc:h2:mem user=SA] rollback-only

외부 트랜잭션 커밋

# 어딘가 누가 롤백 전용으로 표시되어 있으므로 롤백만 가능하도록 설정되어 있음을 인지한다.
Global transaction is marked as rollback-only but transactional code requested commit

Initiating transaction rollback
```
### 정리
- 트랜잭션 매니저는 커밋 시점에 신규 트랜잭션 여부에 따라서 다르게 동작한다.
- 롤백 전용(`rollbackOnly=true`)표시가 있는 지 확인한다. 
  롤백 전용 표시가 있으면 커밋이 아니라 롤백을 한다. 
- `UnexpectedRollbackException`런타임 에러를 발생시킨다.
  시스템 입장에서 롤백이 되었음을 정확하게 알려주기 위함이다.
## 스프링 트랜잭션 전파 - REQUIRES_NEW
- 외부 트랜잭션과 내부 트랜잭션이 각각 별도의 물리 트랜잭션을 갖게 된다.
### 동작 원리
#### 외부 트랜잭션 시작
- 외부 트랜잭션을 시작하면서 `conn0` 를 획득하고 `manual commit`으로 변경해서 물리 트랜잭션을 시작한다.
- 외부 트랜잭션은 신규 트랜잭션이다.(`outer.isNewTransaction()=true` )
#### 내부 트랜잭션 시작
- 내부 트랜잭션을 시작하면서 `conn1` 를 획득하고 `manual commit`으로 변경해서 물리 트랜잭션을 시작한다.
- 기존의 외부 트랜잭션은 보류된다.(실제로는 살아있고 잠깐 보관해두는 것이다.)
- 내부 트랜잭션은 외부 트랜잭션에 참여하는 것이 아니라, `PROPAGATION_REQUIRES_NEW` 옵션을 사용했기 때문에 완전히 새로운 신규 트랜잭션으로 생성된다.(`inner.isNewTransaction()=true` )
#### 내부 트랜잭션 롤백
- 내부 트랜잭션을 롤백한다.
- 내부 트랜잭션은 신규 트랜잭션이기 때문에 실제 물리 트랜잭션을 롤백한다.
- 내부 트랜잭션은 `conn1` 을 사용하므로 `conn1` 에 물리 롤백을 수행 후 커넥션 풀에 반납된다.
#### 외부 트랜잭션 커밋
- 외부 트랜잭션을 커밋한다.
- 외부 트랜잭션은 신규 트랜잭션이기 때문에 실제 물리 트랜잭션을 커밋한다.
- 외부 트랜잭션은 `conn0` 를 사용하므로 `conn0` 에 물리 커밋을 수행한다.
### 주의
- `REQUIRES_NEW`를 사용하면 데이터베이스 커넥션이 동시에 2개 사용된다는 점을 주의해야 한다.

# 스프링 트랜잭션 전파2 - 활용
## 커밋, 롤백
- `LogRepository` 는 트랜잭션와 관련된 `con2` 를 사용한다.
- `로그예외` 라는 이름을 전달해서 `LogRepository` 에 런타임 예외가 발생한다.
- `LogRepository` 는 해당 예외를 밖으로 던진다. 이 경우 트랜잭션 AOP가 예외를 받게된다.
- 런타임 예외가 발생해서 트랜잭션 AOP는 트랜잭션 매니저에 롤백을 호출한다.
- 트랜잭션 매니저는 신규 트랜잭션이므로 물리 롤백을 호출한다.
	- 이전에 유저를 저장할 때 사용한 트랜잭션에는 영향을 주지 않는다.
## 단일 트랜잭션
- 단일 스레드를 사용하면 트랜잭션 매니져는 같은 커넥션을 반환한다.
## 복구 REQUIRED
### 예시 코드
- Commit할 때에는 항상 `rollbackOnly`를 체크함을 확인하는 코드
```Java
@Transactional  
public void joinV2(String username) {  
    Member member = new Member(username);  
    Log logMessage = new Log(username);  
    log.info("== memberRepository 호출 시작 ==");  
    memberRepository.save(member);  
    log.info("== memberRepository 호출 종료 ==");  
    log.info("== logRepository 호출 시작 ==");  
    try {  
        logRepository.save(logMessage);  
    } catch (RuntimeException e) {  
        log.info("log 저장에 실패했습니다. logMessage={}",  
            logMessage.getMessage());  
        log.info("정상 흐름 변환");  
    }  
    log.info("== logRepository 호출 종료 ==");  
}

```
- LogRepository` 에서 예외가 발생한다. 예외를 던지면 `LogRepository` 의 트랜잭션 AOP가 해당 예외를 받는다.
- 신규 트랜잭션이 아니므로 물리 트랜잭션을 롤백하지는 않고, 트랜잭션 동기화 매니저에 `rollbackOnly` 를 표시한다.
- 예외가 `MemberService` 에 던져지고, `MemberService` 는 해당 예외를 복구한다. 그리고 정상적으로 리턴한다.
- 정상 흐름이 되었으므로 `MemberService` 의 트랜잭션 AOP는 커밋을 호출한다.
- 커밋을 호출할 때 신규 트랜잭션이므로 실제 물리 트랜잭션을 커밋해야 한다. 이때 `rollbackOnly` 를 체크한다.
- `rollbackOnly` 가 체크 되어 있으므로 물리 트랜잭션을 롤백한다.
- 트랜잭션 매니저는 `UnexpectedRollbackException` 예외를 던진다.
- 트랜잭션 AOP도 전달받은 `UnexpectedRollbackException` 을 클라이언트에 던진다.