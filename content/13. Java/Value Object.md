---
tags:
  - vo
  - valueObject
---
# Value Object란?
- 불변성 (Immutability)  
	- 생성 후 객체의 상태가 변경되지 않습니다.
- 자기 포함 (Self-contained)
	- 해당 값과 관련된 검증 로직이나 비즈니스 규칙을 스스로 포함합니다.
- 비식별성
	- 동일한 값은 동일한 객체로 간주하며, 별도의 식별자가 필요하지 않습니다.

# 휴대전화 형식에 적용 예시
```Java
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Embeddable
public class UserPhoneNumber {
    public final static String PHONE_NUMBER_PATTERN = "^010-\\d{3,4}-\\d{4}$";

    @Column(name = "phone_number", unique = true, nullable = false)
    private String phoneNumber;

    public UserPhoneNumber(String phoneNumber) {
        if (!StringUtils.hasText(phoneNumber) || !Pattern.matches(PHONE_NUMBER_PATTERN, phoneNumber)) {
            throw new IllegalArgumentException("올바르지 않은 전화번호 패턴입니다.");
        }
        this.phoneNumber = phoneNumber;
    }
}
```
# ValueObject의 비교
## 두 VO를 비교할 때 발생한 문제
PhoneNumber 같은 값 객체(Value Object)를 `List.contains(...)` 등에서 비교하려 했지만 equals와 같은 객체가 없으면 참조(주소)로만 비교가 되기 때문에 같은 전화번화 문자열을 갖고 있는 객체라도 false를 반환하게 된다.   
## 해결방법
아래와 같이 equals와 hashCode 코드를 만들어주면 된다.
```Java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    PhoneNumber that = (PhoneNumber) o;
    return Objects.equals(getPhoneNumber(), that.getPhoneNumber());
}

@Override
public int hashCode() {
    return Objects.hash(getPhoneNumber());
}
```
### 비교 순서
- hashCode() 호출
	- equals를 호출하기 전에 hashCode를 호출해서 같은 hashcode를 갖고 있는지 확인한다.
- equals() 호출
	- 내부 equals 로직을 조사해서 두 객체의 비교를 진행한다.
	- 위의 예시는 클래스 및 내부의 `get`함수를 이용해서 두 객체의 비교를 진행한다.