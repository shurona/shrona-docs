## JdkSerializationRedisSerializer
### 특징
- JdkSerializationRedisSerializer는 기본 값으로 적용 되는 Serializer로 기본 자바 직렬화 방식이다.
-  자바에서 Serializable 인터페이스만 구현하면 별도의 작업 없이 사용할 수 있다.
- Java 직렬화는 객체의 상태와 구조를 완전히 보존하기 때문에, 객체의 정확한 복원이 가능하다
### 단점
- 다른 언어와 호환되지 않는다.
- Java 기본 직렬화는 객체의 클래스 버전에 민감하기 때문에, 객체 구조가 변경되면 역직렬화 시 오류가 발생할 수 있다

## GenericJackson2JsonRedisSerializer
### 특징
- Class Type을 지정해 줄 필요가 없어서 자동으로 객체를 Json 형식으로 직렬화해주는 장점이 있다.
- 다른 언어와 호환이 가능하다.
### 단점
- 직렬화된 데이터가 Class Type을 포함한다.
- 클래스 타입이 포함되므로 저장공간을 많이 차지할 수 있다.

## Jackson2JsonRedisSerializer
### 특징
- class필드를 포함하지 않고 데이터를 Json 형태로 저장할 수 있다. (간결한 데이터 저장)
### 단점
- Class Type 정보를 Serializer에 함께 지정해 줘야 한다.
-  클래스 타입 정보가 포함되지 않기 때문에, 코드 변경 또는 객체 구조 수정 시 타입 불일치로 역직렬화 오류가 발생할 가능성이 있다.
- 역직렬화 시 Generic 타입 지원의 제한

## StringRedisSerializer
### 특징
- String 형식으로 저장되기 때문에 간단하고 빠른 직렬화가 가능하다.
- edis에 저장된 데이터가 문자열 형식 그대로 저장되므로, Redis CLI 등으로 직접 확인할 때 가독성이 좋다.
### 단점
- 숫자, 날짜 등 문자열이 아닌 데이터 타입은 직렬화 전 변환이 필요하다.

