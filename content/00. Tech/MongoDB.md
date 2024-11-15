## 기본 개념
MongoDB는 고성능 고가용성의 NoSQL, Document 기반의 데이터베이스이다.
데이터를 배열 및 중첩 Document와 같은 복잡한 데이터 유형을 효율적으로 저장할 수 있는 유연한 JSON과 유사한 형식인 BSON(Binary JSON)으로 저장합니다.

### Document란
MongoDB의 기본 데이터 저장 단위이다.
document는 BSON 형식에 저장된 필드와 값 쌍으로 저장이 된다.
![[Pasted image 20241106155411.png]]

### Collection 이란
Document를 그룹화한 컨테이너로 관계형 데이터 베이스의 테이블과 유사한 개념이다.
하지만 MongoDB의 컬렉션은 스키마가 고정되어 있지 않아서 각각의 문서가 동일한 필드나 데이터 형식을 가질 필요가 없다.


<img src="../Pasted image 20241106155821.png" alt="Description" width="400" height="300" style="float: left; margin-right: 10000px;" >















### Database 란
컬렉션의 집합으로 특정 애플리케이션이나 프로젝트와 관련된 데이터를 저장하고 관리하는 단위이다.

### Bson이란
Binary Json의 약자로 MongoDB에서 데이터를 저장하고 전송하기 위해 사용하는 이진 형식
JSON 형식과 비슷하지만 더 빠른 처리와 다양한 데이터 타입 지원을 위해 이진화된 형태로 설계되었다.

#### 장점
- 데이터 타입의 유연성: JSON에서 지원하지 않는 Binary Data, Date, ObjectId 등을 BSON은 지원하여, MongoDB와 같은 NoSQL 데이터베이스에서 다양한 데이터를 효율적으로 다룰 수 있습니다.
- 효율적인 저장과 전송: 이진화된 형식이므로 네트워크를 통해 전송할 때 용량을 줄이고 처리 속도를 높일 수 있습니다.
- 구조화된 데이터 관리: BSON은 데이터 크기와 필드 정보를 포함하고 있어 빠른 데이터 접근과 인덱싱을 제공합니다.


## 필드 변경
MongoDB은 NoSQL로 스키마가 정해져 있지는 않지만 Document 단위로 일괄적으로 스키마(필드)를 변경할 수 있다.
format >
`db["databaseName"].updateMany({}, ${아래의 JSON을 추가해준다.})`
예제 >
```Json
db["FoodCart"].updateOne({userId: "master-test"}, 
	{ $set: { "productDto.category": "일식" } })
```
### 새로운 필드 추가
```Json
{
  $set: {
	"newFieldKey": "valueData"
  },
}
```

### Nested 필드 추가
아래와 같은 MongoDB의 Document가 있다고 했을 때 ProductDto에 데이터를 추가하고 싶을 때
```Json
{
  "_id": {
    "$oid": "672c6d0dbf96cb32712ef8a9"
  },
  "userId": "master-test",
  "productDto": {
    "_id": {
      "$numberLong": "2"
    },
    ...
    "description": "수제비 맛있어요",
  },
  "storeId": 2,
}
```
아래와 같은 방식으로 추가가 가능하다
```Json
{ 
	$set: { 
		"productDto.category": "한식" 
	}, 
 }
```

만약 특정 id에만 추가하고 싶을 경우 아래와 같이 진행한다.
```json
db.collection.updateOne(
    { "userId": "master-test" },
    { $set: { "productDto.category": "한식" } }
)
```

### 필드 이름 변경
```Json
{
  $rename: {
	"oldFieldKey": "newFieldKey"
  },
}
```
### 필드 데이터 삭제
```Json
{
  $unset: {
    "deleteField": ""
  }
}
```

#### tag
[[MongoDB]]