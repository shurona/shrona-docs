Spring부트에서는 DataJpaTest어노테이션을 사용하면 간단하게 테스트가 가능하다
H2의 메모리를 사용하므로 Docker를 사용한다거나 하는 식으로 테스트용 데이터베이스를 구축할 필요가 없다.
DataJpaTest는 기본적으로 Jpa에 관련된 Bean만 활성화 되고 Service와 같은 다른 계층은 로드되지 않는다.

# DB를 사용한 인증 Flow

```Java
@DataJpaTest  
class AuthServiceImplTest {  
  
    private final String SECRET_KEY = "secretkey";  
    private final String TOKEN_TIME = "36000000";  
  
    @Autowired  
    private AuthRepository authRepository;  
    @Mock  
    private UserClient userClient;  
    @Mock  
    private RiderClient riderClient;  
    @Mock  
    private RedisUtil redisUtil;  
  
    @Spy  
    private JwtTokenService jwtTokenService = new JwtTokenService(SECRET_KEY, TOKEN_TIME);  
    @InjectMocks  
    private AuthServiceImpl authService;  
  
    @BeforeEach  
    public void setUp() {  
        authService = new AuthServiceImpl(authRepository, userClient, riderClient, redisUtil,  
            jwtTokenService, SECRET_KEY, TOKEN_TIME);  
    }  
  
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