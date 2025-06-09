---
tags:
  - jdbc
---
# JDBC의 이해
## 도입 이유
- 데이터베이스를 다른 종류의 데이터베이스로 변경하면 애플리케이션 서버에 개발된 데이터베이스 사용 코드도 함께 변경해야 한다.
- 개발자가 각각의 데이터베이스마다 커넥션 연결, SQL 전달, 그리고 그 결과를 응답 받는 방법을 새로 학습해야 한다.
## 표준 인터페이스
- JDBC(Java Database Connectivity)는 자바에서 데이터베이스에 접속할 수 있도록 하는 자바 API다. 
- JDBC는 데이터베이스에서 자료를 쿼리하거나 업데이트하는 방법을 제공한다.
### 종류
- `java.sql.Connection` - 연결
- `java.sql.Statement` - SQL을 담은 내용
- `java.sql.ResultSet` - SQL 요청 응답
### 정리
- 애플리케이션 로직은 이제 JDBC 표준 인터페이스에만 의존한다.(추상화에 의존한다.)
  따라서 데이터베이스를 다른 종류의 데이터베이스로 변경하고 싶으면 JDBC 구현 라이브러리만 변경하면 된다. 따라서 다른 종류의 데이터베이스로 변경해도 애플리케이션 서버의 사용 코드를 그대로 유지할 수 있다.
- 개발자는 JDBC 표준 인터페이스 사용법만 학습하면 된다. 한번 배워두면 수십개의 데이터베이스에 모두 동일하게 적용할 수 있다.
## JDBC 연결
- driver가 제공하는 `getConnection`을 사용하면 된다.
```Java
public static Connection getConnection() {  
    try {  
        Connection connection = DriverManager.getConnection(URL, USERNAME, PASSWORD);  
        log.info("get connection {}, class = {}", connection, connection.getClass());  
        return connection;  
    } catch (SQLException e) {  
        throw new RuntimeException(e);  
    }  
}
```
### 과정
- 애플리케이션 로직에서 커넥션이 필요하면 `DriverManager.getConnection()` 을 호출한다.
- `DriverManager` 는 라이브러리에 등록된 드라이버 목록을 자동으로 인식 및 관리하며
  이 드라이버들에게 순서대로 다음 정보를 넘겨서 커넥션을 획득할 수 있는지 확인한다.
- 이렇게 찾은 커넥션 구현체가 클라이언트에 반환된다.
### ResultSet이란
테이블의 형태로 데이터를 관리하는 자료구조

| MEMBER_ID | MONEY |
| --------- | ----- |
| hi1       | 10000 |
| hi2       | 20000 |
| MemberV0  | 10000 |
아래와 같은 cursor를 통해서 데이터를 조회할 수 있다.
```Java
if (rs.next()) {  
    Member member = new Member();  
    member.setMemberId(rs.getString("member_id"));  
    member.setMoney(rs.getInt("money"));  
    return member;  
} else {  
    throw new NoSuchElementException("member not found memberId=" + memberId);  
}
```
