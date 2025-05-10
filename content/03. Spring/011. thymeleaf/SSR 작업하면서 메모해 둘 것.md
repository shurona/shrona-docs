---
tags:
  - thymelaf
---
# Thymelaf(html)에서 페이지로 돌아올 때 데이터를 보존할 수 있을까
Spring MVC와 Thymeleaf에서form 데이터를 사용해서 로직을 처리할 때 오류 발생 시 다시 원본 페이지로 돌려보낼 때 , 기존 HTML에서 데이터를 보전(유지)할 수 있는 방법에 대해 확인해 보았다.
## 사고 과정
- 컨트롤러에서 model에 데이터를 다시 넣어주지 않으면, View에서 해당 리스트는 비어 있게 됩니다.
- HTML(Thymeleaf)만으로는 서버에서 받은 리스트 데이터를 “보전”할 수 없습니다.
- View(HTML)는 서버에서 받은 데이터만을 가지고 렌더링한다.
- 결론
	- 새로고침이나 POST/GET 요청 후에는 컨트롤러가 다시 데이터를 model에 담아서 보내줘야 합니다.
# 변수를 할당해서 사용하기
## 변수 적용 방법
아래와 같이 대괄호를 사용해서 모델에서 값을 갖고 와서 변수에 값을 할당할 수 있다.
```javascript
const roomId = [[${header.id}]];
```
## 실수했던 점
숫자와 같은 경우에는 문제가 없지만 문자열의 경우에는 별도로 따옴표로 감싸줘야 문자열로 처리가 된다.   
감싸주지 않을 경우 변수로 인지해서 ReferenceError가 발생한다.
```Javascript
const username = '[[${header.username}]]';
```

