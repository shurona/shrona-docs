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
<img src="Pasted image 20241106155821.png" alt="Description" width="400" height="300" style="float: left; margin-right: 10000px;">
