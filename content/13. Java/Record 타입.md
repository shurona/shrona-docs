[공식문서]([https://docs.oracle.com/en/java/javase/14/language/records.html](https://docs.oracle.com/en/java/javase/14/language/records.html))
## 개념
Java 14에 추가된 기능으로 새로운 타입 선언자이다. Record는 Enum 처럼 class의 제한된 형태이다. 이것은 이상적인 목적으로 “plain data carriers” 이며 클래스는 변경할 수 없는 데이터와 생성자 및 접근자와 같은 가장 기본적인 메서드만 제공합니다.

만약 객체를 생성할 때 추가적인 작업을 하고 싶으면 생성자에서 약간의 처리를 진행할 수 있다.

### 자동 생성 되는 것
- 각각의 컴포넌트에 대해서 private final 필드 생성
- 컴포넌트의 이름과 같은 public read method가 생성이 된다.
- 각각의 컴포넌트의 목록이 포함되어 있는 public 생성자가 만들어진다
- `equals()`와  `hashCode()` 메소드가 구현이 된다.
- `toString()`메서드가 자동으로 구현이 된다.

## 예시
1. 생성자를 정의할 수 있다.
```Java
record HelloWorld(String message) {
    public HelloWorld {
        java.util.Objects.requireNonNull(message);
    }
}
```
2. DTO에서의 사용
```Java
public record RiderResponseDto(  
    Long riderId,  
    String userId,  
    List<Long> addressCodeList,  
    RiderTransportation transportation,  
    LocalDateTime createdAt  
) {  
  
    public static RiderResponseDto from(Rider rider) {  
        return new RiderResponseDto(rider.getRiderId(), rider.getUserId(),  
            rider.getAddressCodeList(), rider.getTransportation(), rider.getCreatedAt());  
    }  
  
}
```