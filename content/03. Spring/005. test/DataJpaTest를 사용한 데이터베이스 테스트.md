Spring부트에서는 DataJpaTest어노테이션을 사용하면 간단하게 테스트가 가능하다
H2의 메모리를 사용하므로 Docker를 사용한다거나 하는 식으로 테스트용 데이터베이스를 구축할 필요가 없다.
DataJpaTest는 기본적으로 Jpa에 관련된 Bean만 활성화 되고 Service와 같은 다른 계층은 로드되지 않는다.

# DB를 사용한 인증 Flow

```Java
@DataJpaTest  
class AuthServiceImplTest {  
  
	...
	
    @Autowired  
    private AuthRepository authRepository;  
  
    @Test  
    public void 기본_저장확인_테스트() {  
        String userId = "userId";  
        UserRole role = UserRole.USER;  
        ...
        String detailAddress = "detail";  
  
        UserJoinRequest userJoinRequest = new UserJoinRequest(  
            userId, password, ..., detailAddress  
        );  
  
        Auth pwd = authRepository.save(  
            Auth.from(AuthDto.from(userJoinRequest, "pwd", 3377927373L)));  
  
        Auth auth = authRepository.findById(pwd.getId()).orElseThrow();  
  
        Assertions.assertThat(userId).isEqualTo(auth.getUserId());  
  
  
    }  
}
```

`@DataJpaTest`어노테이션을 사용해서 DB 테스트를 진행하고자 하였습니다.
외부로 주입을 해줄 Repository는 `@Autowired`를 사용해서 의존성을 주입해준다.
상관이 없는 클래스는 Mock으로 주입해주고 테스트할 Service 계층을 `@InjectMokck`으로 주입해준다.

# QueryDsl 레포지토리를 테스트
`DataJpaTest`어노테이션은 Jpa와 관련된 빈을 띄워주기 때문에 Spring과 연관이 있는 `@Repository`와 같은 빈은 처리해 주지 않는다.
따라서 QueryDSL과 같이 Spring에서 처리하는 빈은 주입이 되지 않기 때문에 따로 설정을 해줘야 한다.

```Java
@TestConfiguration  
public class QueryDslConfig {  
  
    @PersistenceContext  
    private EntityManager entityManager;  
  
    @Bean  
    public JPAQueryFactory jpaQueryFactory() {  
        return new JPAQueryFactory(entityManager);  
    }  
  
    @Bean  
    public CustomStoreRepository customStoreRepository() {  
        return new CustomStoreRepositoryImpl(jpaQueryFactory());  
    }  
  
  
}
```
위와 같이 `TestConfiguration`어노테이션을 이용해서 Config 설정을 넘겨 줄 수 있다.   
테스트에 사용할 Repository를 빈으로 등록한 다음에 테스트 클래스에 넘겨준다.
```Java
@ExtendWith(MockitoExtension.class)  
@Import(QueryDslConfig.class)  
@DataJpaTest  
class UnitStoreServiceTest {


	@Autowired  
	private CustomStoreRepository customStoreRepository;
	...

}
```
위와 같이 `Import` 어노테이션을 사용해서 위에서 작성한 클래스를 넘겨준 모습을 볼 수 있다.
### JpaAudit 적용
만약 jpaAudit도 사용중인 경우 아래와 같이 어노테이션을 추가해서 설정해 줄 수 있다.
```Java
@EnableJpaAuditing  
@TestConfiguration  
public class QueryDslConfig {
	...
}
```