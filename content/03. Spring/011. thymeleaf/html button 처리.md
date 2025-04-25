---
tags:
  - html
  - button
---
# 버튼 입력 시 처리 방법
## 주어진 button 태그 예시
아래와 같이 버튼하나가 있다고 가정한다.
```html
<button class="btn btn-icon btn-trash" th:data-id="${friend.id}">  
  <i class="fas fa-trash"></i>  
</button>
```
## `document.querySelectorAll`을 사용
querySelectAll을 사용해서 
```javascript
document.querySelectorAll('.btn-trash').forEach(function(btn) {  
  btn.addEventListener('click', function() {  
    console.log('휴지통 버튼이 클릭되었습니다!' + btn.dataset.id);  
  });  
});
```
## button에서 `onclick()` 사용
### `th:onclick` 사용
```html
<button class="btn btn-icon btn-trash"
        th:onclick="|deleteFriend('${friend.id}')|"
		th:data-id="${friend.id}">
  <i class="fas fa-trash"></i>
</button>
```
### `th:attr` 사용
```html
<button class="btn btn-icon btn-trash"
        th:attr="onclick=|deleteFriend('${friend.id}')|"
		th:data-id="${friend.id}">
  <i class="fas fa-trash"></i>
</button>
```
### javascript 파일
위에서 호출할 함수를 script 태그 안에 기록을 한다.
```javascript
function deleteFriend(friendId) {  
  console.log('삭제할 친구 id:', friendId);  
}
```
# button을 눌렀을 때 post 요청
## form을 사용한 요청
```html
<form th:action="@{/friend/v1/banned}" method="post" style="display:inline;">
  <input type="hidden" name="requestId" th:value="${friend.id}" />
  <button type="submit" class="btn btn-icon btn-trash">
    <i class="fas fa-trash"></i>
  </button>
</form>

```
## javascript를 사용해서 요청
```javascript
async function deleteFriend(friendId) {
    const url = '/friend/v1/banned';

    try {
        const response = await fetch(url, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({ friendId: friendId }) // 키 이름 일치
        });

        if (!response.ok) throw new Error('서버 오류');
        
        alert('차단 성공!');
        window.location.reload(); // 페이지 새로고침
    } catch (error) {
        console.error(error);
        alert('차단 실패: ' + error.message);
    }
}

```