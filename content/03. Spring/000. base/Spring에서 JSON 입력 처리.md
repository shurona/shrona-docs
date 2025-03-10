---
tags:
  - json
  - spring/json
---
# 소켓에서 입력 다루기
## 클라이언트에서 String으로 넘어오는 경우
### 넘어오는 데이터 예시 
```Text
const message = JSON.stringify({ userId: 1, chatData: "hello" });
```
### DTO @JsonCreator를 사용한 자동 변환
ObjectMapper를 사용해서 String을 json으로 변환한다.
```Java
@JsonCreator  
public static ChatRequestDto fromJson(String json) {  
    try {  
        ObjectMapper objectMapper = new ObjectMapper();  
        return objectMapper.readValue(json, ChatRequestDto.class);  
    } catch (Exception e) {  
        throw new RuntimeException("JSON 변환 오류: " + json, e);  
    }  
}
```

### Controller에서 처리하기
ObjectMapper를 사용해서 String을 json으로 변환한다.
```Java
@MessageMapping("/chat")
@SendTo("/topic/chat")
public ChatRequestDto handler(String message) {
    ObjectMapper objectMapper = new ObjectMapper();
    ChatRequestDto chatRequestDto = objectMapper.readValue(message, ChatRequestDto.class);
    return chatRequestDto;
}
```

## 클라이언트에서 JSON 형태로 넘어 오는 경우
받을 형식을 아래와 같이 형식에 맞춰서 넘어 오는 경우 크게 설정 없이 자동으로 변환될 수 있다.
클라이언트 예시
```javascript
const message = { userId: 1, chatData: "hello" };

stompClient.publish({ destination: destinationPath, body: JSON.stringify(message) });
```
서버 DTO 예시
```Java
public record ChatRequestDto(
    Long userId,
    String chatData
) {}
```


