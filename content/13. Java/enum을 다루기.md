---
tags:
  - enum
---
# 괄호를 이용한 ENUM 생성자 처리
## 예시 및 설명
Java에서 아래와 같이 괄호로 값을 전달하는 방식은 Enum 생성자(enum constructor) 호출 방식이다.
```Java
public enum FriendRequest {
    REQUESTED(RequestStatus.REQUEST),
    REFUSED(RequestStatus.REFUSED);
}
```
이 방식은 Enum 상수 각각이 생성자에 값을 전달하여 내부 필드를 초기화하는 패턴이다.   
각 상수(REQUESTED, REFUSED)는 Enum이 정의한 생성자에 지정된 값을 넘긴다.   
이렇게 하면 ENUM상수 마다 다른 값을 갖게 될 수 있다.   

위와 같이 괄호를 이용해서 처리를 하려면 아래와 같이 생성자와 변수가 있어야 한다.
```Java
private final String status;
private FriendRequest(String status) {
    this.status = status;
}
```
## 완성된 예시 코드
```Java
@RequiredArgsConstructor  
@Getter  
public enum FriendRequest {  
    REQUESTED(RequestStatus.REQUEST),  
    REFUSED(RequestStatus.REFUSED);  
  
    private final String status;  

	// 상태 정보를 static String으로 저장한다.
    public static class RequestStatus {  
        public static final String REQUEST = "FRIEND_REQUEST";  
        public static final String REFUSED = "FRIEND_REFUSED";  
    }  
}
```