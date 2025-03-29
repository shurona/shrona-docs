---
tags:
  - aop
---
# AOP를 사용한 페이지네이션

## AOP를 통해서 부가기능을 모듈화

1. @Aspect
    1. Sprinig 빈 클래스에만 적용이 가능하다
2. 어드바이스 종류
    1. Around(After + Before)
    2. Before
    3. After
    4. AfterRunning
    5. AfterThrowing

## 포인트컷

- 포인트컷 Expression Languare

포인트 컷을 사용해서 적용 될 위치를 지정할 수 있다.
### AOP를 적용하기 위한 위치 지정
#### 위치를 지정해서 적용
```java
@Pointcut("execution(* com.domain_expansion.integrity.slack.presentation.*.*(..))")
```
#### annotation을 생성해서 원하는 위치로 적용
아래와 같이 임의의 annotation을 만든 다음에 해당 위치에 미리 넣어놓는다.

```java
  @DefaultPageSize
  @GetMapping
  public SuccessResponse<?> findSlackMessageList(
      @PageableDefault(value = 10, size = 10, page = 0) Pageable pageable
  ) {
	  ...
  }
```

### AOP의 구현

```java
@Pointcut("@annotation(com.domain_expansion.integrity.slack.common.aop.DefaultPageSize)")
private void defaultPageSize() {
}

@Before("defaultPageSize()")
public Object checkDefaultPage(ProceedingJoinPoint joinPoint) throws Throwable {

    Object[] args = joinPoint.getArgs();

    System.out.println("출력은 어떻게 되는 거임 : " + Arrays.toString(args));

    return joinPoint.proceed();

}
```