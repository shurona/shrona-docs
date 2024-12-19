---
tags:
  - validation
---

# Input Validation 처리
## Request Body에서 접근 제어

### Request body가 없는 경우

GET이 아닌 일반적인 요청에서 Body는 자주 사용이 된다. 하지만 Body가 필요함에도 불구하고 client에서 의도적으로 Body를 삭제하면 아래와 같이 500에러가 발생하였다.

```text
org.springframework.http.converter.HttpMessageNotReadableException: 
Required request body is missing:
```

이를 제어해서 400에러로 변환하기 위해 방법을 알아보았다.
### 해결 방법

1. `Required=false` 처리

Request를 받는 Controller에서 Required를 false로 처리한 후에 null인 경우에 예외처리를 진행해준다.

2. 글로벌 `ExceptionHandler`로 처리

Request 요청을 받을 때 Body가 잘못된 경우 아래와 같은 `HttpMessageNotReadableException`에러를 전달해준다.

```java
@ExceptionHandler(HttpMessageNotReadableException.class)
public ResponseEntity<? extends CommonResponse<?>> handleHttpMessageNotReadableException(HttpMessageNotReadableException exception) {
    
    return ResponseEntity
        .status(HttpStatus.BAD_REQUEST)
        .body(CommonResponse.error(INVALID_INPUT.getStatus(),
            "RequestBody가 잘못 입력 되었습니다.", exception.getMessage()));
}
```

위의 에러는 null인 경우 이외에도 잘못된 Body 데이터가 들어온 경우에도 해당 Handler를 통하게 된다.
개별적으로 잘못된 에러를 주기 위해서는 추가적인 처리를 진행하고자 한다.

### DateTime 변환 문제
LocalDateTime으로 변환을 할 때 원치 않은 Input을 넣어줄 경우 아래와 같은 에러가 발생을 한다.
Input 예제 `2024-10-212`
```text
JSON parse error: Cannot deserialize value of type `java.time.LocalDate` from String \"2024-10-212\": Failed to deserialize java.time.LocalDate: (java.time.format.DateTimeParseException) Text '2024-10-212' could not be parsed, unparsed text found at index 10
```

DateTime으로 변환을 할 때 `DateTimeParseException`의 에러가 발생을 함으로 해당 에러에 대한 핸들러를 Controller내부에 아래와 같이 적용하였다.

DateTime을 파싱하는 경우는 서버 모듈마다 다를 수 있다고 판단해서 각 Controller에 직접 구현하도록 하였다.
```Java
@ExceptionHandler(DateTimeParseException.class)
public ResponseEntity<? extends CommonResponse<?>> dateTimeInputError(
    DateTimeParseException exception) {

    return ResponseEntity.status(BAD_REQUEST)
        .body(CommonResponse.error(BAD_REQUEST, "잘못된 날짜 형식 입니다. yyyy-MM-dd 형식으로 입력해주세요",
            exception.getMessage()));
}
```

### Enum 타입 변환 문제
`RequestBody`에서 Enum을 입력 받을 때 아래와 같이 Enum을 기본으로 입력을 받으려 하였다.
```Java
@NotNull  
DateType dateType,
```
하지만 이렇게 할 경우 원치 않은 입력이 들어올 경우 아래와 같이 에러가 발생을 하게 된다.
```text
HttpMessageNotReadableException: JSON parse error
```

위의 에러를 처리 하기 위해서 `GlobalExceptionHandler`에서 처리를 할 경우 프론트에 원하는 에러를 전달해 주는데 어려움이 있었다. 
### 해결 방법
1. 직렬화 하는 과정에서 문제가 발생하므로 아래와 같이 직렬화 로직을 구현해서 적용하려 하였다.
```java
public class DiscountTypeDeserializer extends JsonDeserializer<DiscountType> {

    @Override
    public DiscountType deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {
        try {
            return DiscountType.valueOf(p.getText().toUpperCase());
        } catch (IllegalArgumentException e) {
            throw new CouponException(CouponErrorCode.INVALID_INPUT);
        }
    }
}

@JsonDeserialize(using = DiscountTypeDeserializer.class)
```

하지만 위와 같이 직렬화 에러를 처리하였음에도 불구하고  `HttpMessageNotReadableException`에러가 위에서 발생한 에러를 감싸서 결과는 `HttpMessageNotReadableException`로 처리가 되는 문제는 유지되었다.

2.  Annotation을 사용해서 처리
`RequestBody`에서 Enum으로 받는 것 대신 String으로 받게 변경을 한다.
또한 Custom Annotation의 target으로 Enum class를 지정해준다.
[참고 공식 문서](https://docs.spring.io/spring-framework/reference/core/validation/beanvalidation.html#validation-beanvalidation-spring-inject)

```java
@EnumValidation(target = DateType.class)  
@NotNull  
String dateType
```

Annotation 설정
```Java
@Constraint(validatedBy = {EnumValidator.class})
@Target({FIELD, ANNOTATION_TYPE, PARAMETER, TYPE_USE})  
@Retention(RUNTIME)  
public @interface EnumValidation {  

	 // 에러가 발생했을 때 전달해줄 Message
    String message() default "{invalid.enum.value}";  
  
    Class<?>[] groups() default {};  
  
    Class<? extends Payload>[] payload() default {};  
  
    Class<? extends Enum<?>> target();  
}
```


#### EnumValidator 클래스 설정

검증할 Enum 정보를 먼저 저장한다.

```java
Enum<?>[] enumConstants = this.validation.target().getEnumConstants();  
```

입력받은 값 중에 Enum이 있는지 확인한다.
```java
boolean contains = Arrays.stream(enumConstants).map(Enum::name)  
        .anyMatch(name -> name.equals(input));
```

만약 포함되어 있지 않다면 아래와 같이 어떤 Enum이 필요한지 프론트에 알려주기 위한 준비 후 에러를 전달해 준다.
```java
if (!contains) {  
        String allowedValues = Arrays.stream(enumConstants)  
            .map(Enum::name)  
            .collect(Collectors.joining(", "));  
  
        // 동적 메시지를 설정  
        context.disableDefaultConstraintViolation();  // 기본 메시지 비활성화  
        context.buildConstraintViolationWithTemplate(  
                String.format("입력값 '%s'은 올바르지 않습니다. 허용되는 값: [%s]", input, allowedValues))  
            .addConstraintViolation();  
    }  
```

#### 최종 코드
```java
public class EnumValidator 
	implements ConstraintValidator<EnumValidation, String> {  
  
    private EnumValidation validation;  
  
    @Override  
    public void initialize(EnumValidation constraintAnnotation) {  
        this.validation = constraintAnnotation;  
    }  
  
    @Override  
    public boolean isValid(String input, ConstraintValidatorContext context) {  
        if (input == null || input.isBlank()) {  
            return true;  
        }  
  
        Enum<?>[] enumConstants = this.validation.target().getEnumConstants();  
          
        boolean contains = Arrays.stream(enumConstants).map(Enum::name)  
            .anyMatch(name -> name.equals(input));  
  
        if (!contains) {  
            String allowedValues = Arrays.stream(enumConstants)  
                .map(Enum::name)  
                .collect(Collectors.joining(", "));  
  
            // 동적으로 에러 메시지를 설정  
            context.disableDefaultConstraintViolation();  // 기본 메시지 비활성화  
            context.buildConstraintViolationWithTemplate(  
                    String.format("입력값 '%s'은 올바르지 않습니다. 허용되는 값: [%s]", input, allowedValues))  
                .addConstraintViolation();  
        }  
  
        return contains;  
    }  
}
```

## Url 파라미터에서 문제가 발생한 경우
### Resource Not Found Exception
잘못된 Path가 입력되었을 때 위와 같은 에러가 발생을 하기에 전부 404를 하기 전에 한 번 찾는 과정을 확인해 보았다.
Dispatcher Servlet에서 아래의 메서드로 들어가서 먼저 Handler를 찾게 된다.

```java
class위치 => DispatcherServlet.class

protected void doDispatch(HttpServletRequest request,   
  HttpServletResponse response) throws Exception
```

아래 그림과 같은 hanlderMapping목록을 순차적으로 돌면서 해당하는 Handler를 찾는다.

```java
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

> 참고로 존재하는 path로 받은 request는 RequestMappingHandler에서 찾는다.

![[Pasted image 20241102205041.png]]

이후 존재하지 않는 경우 해당 함수 내부의 아래를 호출함으로써 exception이 존재함을 알려준다.

```java
processDispatchResult(
	processedRequest, response, mappedHandler, mv, dispatchException);
```

마지막으로 리소스의 위치를 찾는 순서
1. class path resource [META-INF/resources/]
2. class path resource [resources/]
3. class path resource [static/]
4. class path resource [public/]
5. ServletContext resource [/]