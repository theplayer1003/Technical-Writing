# H2 데이터베이스와 JDBC

## 목표
- 가벼운 H2 데이터베이스를 활용해 Java 애플리케이션과 데이터베이스를 연결한다
- JDBC 표준 인터페이스를 사용해 CRUD 를 구현한다
- `try-with-resources` 구문으로 자원을 안전하게 관리한다

## 사전 요구사항
- JDK 8 이상 (H2 2.x 버전)
- JDK 17 이상 권장 (LTS 버전, JDK 21 도 가능, Spring Boot 3.x 호환)
- H2 Database 설치 및 실행 가능 상태

## 단계별 가이드
### 1. H2 데이터베이스 연결 설정
MySQL 의 복잡한 설치 세팅보다 가볍게 실행 가능한 H2 데이터베이스가 학습에 효율적이다. 이를 통해 JDBC 동작 원리에 집중해서 학습한다.

```java
private static final String DB_URL = "jdbc:h2:file:./data/missiondb";  
private static final String USER = "sa";  
private static final String PASS = "";

public static void main(String[] args) { 
	try (Connection conn = DriverManager.getConnection(DB_URL, USER, PASS)) { 
		System.out.println("H2 데이터베이스 연결 성공: " + conn); 
	} catch (SQLException e) { 
	e.printStackTrace(); 
	} 
}
```

- `DriverManager.getConnection()`
	- `DriverManager` 클래스가 `getConnection` 메서드로 여러 드라이버들을 스캔하며, `jdbc:h2` 를 처리할 수 있는 드라이버를 찾아 연결한다.
	- H2 드라이버가 실제 DB와 연결하고 `Connection` 인터페이스 구현체를 생성해 반환한다.

### 2. 테이블 생성, DDL
연결된 `Connection` 객체를 통해 SQL 을 전송한다. 정적인 쿼리 실행에는 `Statement` 인터페이스를 사용할 수 있다.

```java
// try-with-resources 구문 내부에서 Statement 생성
try (Statement stmt = conn.createStatement()) {  
    String createTableSql = "CREATE TABLE IF NOT EXISTS USERS " +  
            "(id INT AUTO_INCREMENT PRIMARY KEY, " +  
            " username VARCHAR(255) NOT NULL, " +  
            " email VARCHAR(255))";  
  
    stmt.executeUpdate(createTableSql);  
    System.out.println("테이블 USERS 생성 쿼리 완료");  
}
```

- `conn.createStatement()`
	- `Connection` 객체로부터 정적 SQL 전송 객체인 `Statement` 인터페이스 구현체를 생성한다
- `stmt.executeUpdate(createTableSql)`
	- `Statement` 객체를 통해 DB 상태를 변경하는 쿼리를 실행한다

### 3. 데이터 삽입, 동적 쿼리
동적 쿼리로 데이터를 삽입 할때는 `PreparedStatement` 를 사용한다. `?`(Placeholder) 를 사용하여 값을 동적으로 바인딩한다.

```java
String insertSql = "INSERT INTO USERS (username, email) VALUES (?, ?)";  
try (PreparedStatement pstmt = conn.prepareStatement(insertSql)) {  
  
    pstmt.setString(1, "pobi");  
    pstmt.setString(2, "pobi@nate.com");  
    pstmt.executeUpdate();  
  
    pstmt.setString(1, "james");  
    pstmt.setString(2, "james@budybudy.com");  
    pstmt.executeUpdate();  
  
    System.out.println("데이터 2건 삽입 완료");  
}
```

- `pstmt.setString(1, "pobi")`
	- `set...` 메서드를 통해 쿼리의 `?` 순서대로 값을 바인딩한다
- `pstmt.executeUpdate(createTableSql)`
	- `PreparedStatement` 객체를 통해 DB 상태를 변경하는 쿼리를 실행한다

### 4. 데이터 조회, ResultSet
저장된 데이터를 조회하기 위해 `executeQuery()` 를 사용하며, 결과는 `ResultSet` 객체로 반환한다

```java
String staticSelectSql = "SELECT * FROM USERS";  
try (Statement stmt = conn.createStatement();  
     ResultSet rs = stmt.executeQuery(staticSelectSql)) {  
  
    System.out.println("\n-- USERS 테이블 데이터 조회 --");  
    while (rs.next()) {  
        System.out.printf("ID: %d, Username: %s, Email: %s\n",  
                rs.getInt("id"),  
                rs.getString("username"),  
                rs.getString("email")  
        );  
    }  
}
```

- `stmt.executeQuery(staticSelectSql)`
	- `executeQuery(query)` 메서드를 통해 DB로부터 조회를 수행한다
- `rs.next()`
	- 결과 데이터 집합을 담고 있는 객체를 순회하며 값을 읽어온다
	- 데이터가 존재하면 `true` 를 반환하고 커서를 다음으로 이동시킨다

### try-with-resources
JDBC 프로그래밍에서는 사용한 자원(`Connection`, `Statement`, `ResultSet`) 을 반드시 반납(`close`) 해야한다. 그렇지 않으면 메모리 누수나 DB 연결 고갈 문제가 발생할 수 있다.

Java 7 부터 도입된 `try-with-resources` 구문을 사용해 `finally` 블록 없이도 자원 해제가 가능하다.

```java
try (Connection conn = DriverManager.getConnection(DB_URL, USER, PASS)) {
	...
} catch (SQLException e) {
	...
}
```

`try-catch-finally` 구문은 `finally` 블록의 `close()` 메서드에서 또 다른 예외가 발생하면 최초 발생한 예외가 덮어씌워져 사라지는 문제가 있다.
`try-with-resources` 구문은 이를 해결하기 위해, 후속 예외를 버리지 않고 원본 예외의 억제된 예외(Suppressed Exception) 목록에 추가하여 함께 던진다.