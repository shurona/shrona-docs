---
tags:
  - error
  - html
---
# 에러 처리 
## 클라이언트 요청
- javascript에서 클라이언트에서 요청을 보낸다.
```javascript
fetch(`url`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ phone: newPhone })
})
.then(async (response) => { // 동기를 통해서 response를 처리 한다.
if (!response.ok) {
  const error = await response.json();
  throw new Error(error.message || '서버 에러'); // 에러를 전달
}
  td.innerText = newPhone;
  td.setAttribute('data-phone-number', newPhone);
})
.catch(error => { // 에러가 발생했을 때 잡아서 처리
  alert('변경 중 에러 발생 : ' + error.message);
  td.innerText = beforePhone;
});
```
## 서버에서 에러 전달
아래와 같이 ResponseDto로 에러를 반환해준다.
```Java
public class ErrorResponseDto {    
    private String message;  
}
```