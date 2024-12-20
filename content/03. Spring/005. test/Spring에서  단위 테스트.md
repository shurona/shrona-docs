---
tags:
  - spring
  - junit
  - test
  - mockito
---
## 사용하는 라이브러리
- Junit5
	- 자바 단위 테스트를 위한 테스트 프레임워크
- AssertJ
	- 자바 테스트를 돕기 위해 다양한 문법을 지원하는 라이브러리

## given when then

- given
	- 데이터의 준비 과정
- when
	- 주어진 데이터를 기반으로 동작을 진행했을 때
- then
	- 동작 실행에 대한 Output 데이터 검증 부분

예시 - 퀴즈 데이터를 생성하는 것을 검증하기 위한 테스트이다.
```Java
// given  
User user = new User("nickname", "loginId", "password");  
this.userRepository.save(user);  

// 단어와 word_user 저장  
for (int i = 0; i < 20; i++) {  
	Word word = new Word("word" + i, "meaning");  
	this.wordRepository.save(word);  
	joinWordRepository.saveUserWord(user, word);  
}  

// when  
Long quizSetId = this.quizService.generateQuizSet(user.getId());  
QuizSet quizInfo = this.quizService.getQuizInfo(quizSetId);  

// then  
assertThat(quizInfo.getCurrentSequence()).isEqualTo(0);  
assertThat(quizInfo.getQuizDetails().size()).isEqualTo(10);  
assertThat(quizInfo.getUser().getId()).isEqualTo(user.getId());
```

## Mockito
Mockito는 자바 언어를 사용하는 소프트웨어 개발자들이 단위 테스트를 작성하는 데 사용하는 오픈 소스 프레임워크이다. 
Mock이 필요한 테스트에 직관적으로 사용할 수 있도록 만들어졌습니다.

### 주요 어노테이션
- Mock
	- Mock 객체를 생성할 때 사용한다.
- MockBean
	- Spring에서 제공하는 Bean을 Mocking할 때 사용한다.
- Spy
	- 실제 객체를 사용해서 테스트를 진행한다.
	- Stub 하지 않은 메소드들은 원본 그대로 사용한다.
- InjectMocks
	- 주입할 대상 객체의 생성자를 호출하여 모의 객체를 자동으로 주입한다.


### Stub이란
테스트 도중에 호출 될 메서드 들에 대해서 미리 답변을 준비해 놓는 것을 의미한다.   
즉 mock객체의 메소드를 호출했을 때 어떤 값을 리턴할 지 미리 정해놓는 것이다.

#### 주요 메서드
- doReturn()
	- 특정 값을 반환해야 하는 경우
- doNothing()
	- 아무 값도 반환하지 않는 경우
- doThrow()
	- 예외를 발생시키는 경우

예제
```Java
CompanyResponseDto companyResponseDto = mock(CompanyResponseDto.class);

when(companyResponseData.getData()).thenReturn(companyResponseDto);

doReturn(new CommonResponse<RiderResponseDto>(200, "메시지", givenRider))  
    .when(riderClient).authRider(requestDto);
```

#### 두 방식의 차이점

| 특징        | when(...).thenReturn(...) | doReturn(...).when(...)               |
| --------- | ------------------------- | ------------------------------------- |
| 기본 사용 사례  | 일반적인 메서드 Stub 처리          | 예외를 방지하거나 final/static/private 메서드 처리 |
| 가독성       | 상대적으로 간결하고 직관적            | 약간 복잡하고 이해하기 어려울 수 있음                 |
| 예외 발생 가능성 | 실제 메서드 호출 시 예외 발생 가능      | 메서드를 호출하지 않으므로 예외 발생하지 않음             |
| 사용 가능 범위  | 일반 메서드                    | final, private, void 메서드에도 사용 가능      |
| 실행 시점     | 메서드가 실제 호출될 때 Stub 설정     | Stub 설정 시점에 메서드를 실행하지 않음              |
### Spring에서 사용 방법
아래와 같이 테스트 할 클래스의 어노테이션으로 붙여주면 된다.
```Java
@ExtendWith(MockitoExtension.class)  
class 테스트_클래스_이름 {
	...
}
```

### 공통 적용 사항
테스트를 원하는 RiderAuthServic는 InjectsMock으로 주입해준다.   
외부 서비스를 활용해야 하는 RiderClient 및 RedisUtils는 Mock을 이용한다.  
다른 외부 서비스를 사용하지 않는 JwtTokenService는 Spy를 써서 기존 로직을 이용한다.  
Inject될 Service에 생성자로 추가해줘야 할 데이터가 있어서 `BeforeEach`를 활용해서 셋업을 해준다.
```Java
@InjectMocks  
private RiderAuthService riderAuthService;  
  
@Mock  
private RiderClient riderClient;  
  
@Mock  
private RedisUtil redisUtil;  
  
@Spy  
private JwtTokenService jwtTokenService = new JwtTokenService(  
    "dGVzdE1vY2t0ZXN0TW9ja3Rlc3RNb2NrdGVzdE1vY2t0ZXN0TW9ja3Rlc3RNb2Nr", "3600");  
 
@BeforeEach  
void setUp() {  
    // `@Value`로 주입되는 값은 생성자로 전달  
    riderAuthService = new RiderAuthService(riderClient, redisUtil, jwtTokenService, "3600");  
  
}
```

#### given when then을 사용해서 작성한 성공 예제
라이더인 유저가 정상적으로 인증을 성공하는지 테스틑 하는 Flow이다.
서비스의 입력값과 내부로직 상 돌아갈 데이터를 Mock해서 given으로 처리해준다.
외부 호출을 진행하지 않을 2개의 메서드를 Mocking 처리 해준다.
이후 생성된 token값과 내부 함수 호출을 점검한다.
```Java
@DisplayName("라이더 인증 서비스 성공")  
@Test  
public void 인증_성공_테스트() {  

	String userId = "userId";  
	RiderAuthRequestDto requestDto = new RiderAuthRequestDto(userId, "password");  
	RefreshTokenDto refreshTokenDtoMock = mock(RefreshTokenDto.class);  

	// given  
	RiderResponseDto givenRider = new RiderResponseDto(  
		1L, userId, Arrays.asList(1L, 2L), RiderTransportation.MOTORCYCLE,  
		LocalDateTime.now());  

	doReturn(new CommonResponse<RiderResponseDto>(200, "메시지", givenRider))  
		.when(riderClient).authRider(requestDto);  

	doNothing().when(redisUtil).setDataRefreshToken(any(RefreshTokenDto.class));  

	// when  
	String token = riderAuthService.riderAuth(requestDto);  
	Claims claims = jwtTokenService.extractClaims(token);  

	// then  
	// token의 subject 확인  
	Assertions.assertThat(claims.getSubject()).isEqualTo(userId);  

	// 내부 함수 호출 횟수 확인  
	verify(redisUtil, times(1)).setDataRefreshToken(any(RefreshTokenDto.class));  

}
```

#### 에러 처리 테스트
1. assertThatThrownBy 사용

특정 메서드에 대해서 에러가 발생시키는 Mock을 만든다.
이후  ssertThatThrownBy를 사용해서 클래스의 타입과 메세지를 확인할 수 있다.
```Java
doThrow(new FeignException.NotFound("Rider not found", request, null, null)).when(  
    riderClient).authRider(requestDto);  
  
// then  
ssertThatThrownBy(() -> {  
	String token = riderAuthService.riderAuth(requestDto);  
})
.isInstanceOf(AuthException.class)  
.hasMessage(AuthErrorCode.USER_NOT_FOUND.getMessage());
```


2. catchThrowable 사용   

catchThrowable을 사용해서 에러를 먼저 받은 다음에 기존과 같은 방식으로 인스턴스와 메시지를 비교해준다.
```Java
doThrow(new FeignException.BadRequest("Invalid Password", request, null, null)).when(  
    riderClient).authRider(requestDto);  
  
Throwable error = Assertions.catchThrowable(() -> {  
    String token = riderAuthService.riderAuth(requestDto);  
});  
  
// then  
Assertions.assertThat(error)  
    .isInstanceOf(AuthException.class)  
    .hasMessage(AuthErrorCode.INVALID_PASSWORD.getMessage());
```

3. assertThatExceptionOfType

Type을 사용해서 먼저 타입을 알려준 다음에 에러 발생 시 메시지 확인도 가능하다.
```Java
assertThatExceptionOfType(AuthException.class).isThrownBy(() -> {  
    String token = riderAuthService.riderAuth(requestDto);  
})
.withMessage(USER_NOT_FOUND.getMessage());
```