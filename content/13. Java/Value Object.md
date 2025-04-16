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