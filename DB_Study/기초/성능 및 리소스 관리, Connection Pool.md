# getConnection 은 비용이 많이 든다
- 지금까지 별거 아닌 거처럼 계속 `Connection` 을 생성해왔지만
- 사실 이건 비용이 매우 비싸다
	1. TCP/IP 소켓 열기
	2. DB 인증
	3. 세션 설정, 메모리 공간 리소스 할당 해야함
	4. `Connection` 객체 생성 및 반환
- 어느정도로 비싸냐면 쿼리 한번 실행하는 것에 수천 배 정도 비싸다

- 그래서 나온 해결책이 바로 `Connection Pool`
- 데이터베이스 커넥션을 미리 일정 개수만큼 생성해서 `Pool` 에 보관해두고, 필요할 때마다 빌려 쓰고 반납하자
- `conn.close()`
	- `DriverManger` 의 `close()` 는 DB와의 물리적 연결이 끊어지는 것
	- `Connection Pool` 의 `close()` 는 풀에서 빌려온 커넥션을 반납, 물리적 연결은 유지되는 상태
- `Connection Pool` 은 애플리케이션 시작과 함께 `Connection` 을 미리 생성해서 DB와 연결해 둔다
- 이제 사용자 요청이 들어오면 `DriverManager` 가 아닌 `Connection Pool` 에게 요청한다
- `Connection Pool` 은 이미 만들어둔 커넥션 하나를 반환하고 '대여 중' 으로 표시한다
- 사용자는 이를 사용해 쿼리를 수행하고 `conn.close()` 를 호출한다
- `Connection Pool` 은 '대여 중' 표시를 '대기 중' 상태로 바꾼다
- 반복

- 이를 통해 '연결, 인증' 비용을 프로그램 시작 처음에만 발생시키고 DB가 감당 가능한 최대 연결 수를 제어할 수 있게 된다

# DataSource 인터페이스
- `DriverManager` 인터페이스 대신 쓸 인터페이스가 `DataSource` 다
- 이제 `Connection` 얻는 법을 추상화 해 개발자는 무슨 라이브러리에 대한 구현체인지 알 필요가 없다
- 커넥션 풀 에는 `HikariCP`, `Tomcat-DBCP` 등이 있다
- `HikariCP` 가 `Spring Boot` 의 기본 값이라고 하니 이를 사용해보자

```java
package practiceH2;  
  
import com.zaxxer.hikari.HikariConfig;  
import com.zaxxer.hikari.HikariDataSource;  
import java.sql.Connection;  
import java.sql.PreparedStatement;  
import java.sql.ResultSet;  
import java.sql.SQLException;  
import javax.sql.DataSource;  
  
public class ConnectionPoolSample {  
  
    private static final DataSource dataSource = createDataSource();  
  
    public static void main(String[] args) {  
        setupDataBase();  
  
        String sql = "SELECT * FROM ACCOUNTS";  
  
        try (Connection conn = getConnection();  
             PreparedStatement pstmt = conn.prepareStatement(sql);  
             ResultSet rs = pstmt.executeQuery()) {  
  
            System.out.println("-- ACCOUNTS 테이블 조회 (Connection Pool) --");  
            while (rs.next()) {  
                System.out.printf("ID : %d, Name : %s, Password : %s\n",  
                        rs.getInt("id"),  
                        rs.getString("username"),  
                        rs.getString("password"));  
            }  
        } catch (SQLException e) {  
            System.err.println("DB 조회 중 오류 발생 (CP)");  
            e.printStackTrace();  
        }  
    }  
  
    private static void setupDataBase() {  
  
        String dropSql = "DROP TABLE IF EXISTS ACCOUNTS";  
        String createSql = "CREATE TABLE ACCOUNTS (" +  
                "id INT AUTO_INCREMENT PRIMARY KEY, " +  
                "username VARCHAR(255) NOT NULL, " +  
                "password VARCHAR(255) NOT NULL)";  
        String insertSql = "INSERT INTO ACCOUNTS (username, password) VALUES ('james', 'pobi')";  
  
        Connection conn = null;  
        try {  
            conn = getConnection();  
  
            conn.setAutoCommit(false);  
  
            try (PreparedStatement pstmt = conn.prepareStatement(dropSql)) {  
                pstmt.executeUpdate();  
            }  
  
            try (PreparedStatement pstmt = conn.prepareStatement(createSql)) {  
                pstmt.executeUpdate();  
            }  
  
            try (PreparedStatement pstmt = conn.prepareStatement(insertSql)) {  
                pstmt.executeUpdate();  
            }  
  
            conn.commit();  
            System.out.println("\n-- DB 준비 트랜잭션 성공 --");  
  
        } catch (SQLException e) {  
            System.out.println("\n-- DB 준비 트랜잭션 실패, 롤백 --");  
            try {  
                if (conn != null) {  
                    conn.rollback();  
                }  
            } catch (SQLException e2) {  
                e2.printStackTrace();  
            }  
            e.printStackTrace();  
        } finally {  
            if (conn != null) {  
                try {  
                    conn.setAutoCommit(true);  
                    conn.close();  
                } catch (SQLException e) {  
                    e.printStackTrace();  
                }  
            }  
        }  
    }  
  
    private static Connection getConnection() throws SQLException {  
        return dataSource.getConnection();  
    }  
  
    private static DataSource createDataSource() {  
        HikariConfig config = new HikariConfig();  
  
        config.setJdbcUrl("jdbc:h2:file:./data/practicedb_connectionpool");  
        config.setUsername("sa");  
        config.setPassword("");  
  
        config.setMaximumPoolSize(10);  
        config.setConnectionTimeout(30000);  
        config.setPoolName("MyH2Pool");  
  
        return new HikariDataSource(config);  
    }  
}
```

- `HikariCP` 는 `DataSource` 의 구현체이다
- 핵심 흐름은
	1. 설정 객체를 만들고
	2. `DataSource` 를 생성하고
	3. `getConnection()` 을 호출한다
- 먼저 설정을 담당하는 `HikariConfig` 객체를 생성해야한다
- 제일 먼저 필수정보인 DB 접속 정보 (ULR, User, Password) 를 설정한다
- `config.set...` 시리즈로 설정 가능
- 이제 옵션을 설정할 차례다
- 최대 커넥션 (제일 중요한 성능 관련 세팅이다)
	- `config.setMaximumPoolSize(10);`
- 유휴 커넥션 세팅
	- `config.setMinimumIdel(2);`
- 타임아웃 세팅
	- `config.setConnectionTimeout(30000);` (ms 단위)
- 풀에서 유휴 상태로 머무는 시간
	- `config.setIdleTimeout(600000);` (ms 단위)
- 로그 식별용 이름 세팅
	- `config.setPoolName("MyHikariPool");`

- 실무에서는 설정과 코드를 분리하기 위해 `Porperties` 파일을 설정한다
	- `src/main/resources/hikari.properties`
```java
# 필수 설정 (DataSource 속성)
jdbcUrl=jdbc:h2:file:./data/mydb;AUTO_SERVER=TRUE
username=sa
password=

# 풀 옵션 (HikariCP 고유 속성)
maximumPoolSize=10
minimumIdle=2
connectionTimeout=30000
idleTimeout=600000
poolName=MyHikariPool

# 자바 코드에서는 경로만 지정해주면 된다
HikariConfig config = new HikariConfig("/hikari.properties");
```

# Singleton 으로 관리
```java
private static final DataSource dataSource = createDataSource();

new HikariDataSource(config);
```

- 실제 물리적 DB 와 커넥션을 맺는 작업이기 때문에 애플리케이션 시작 후 단 한 번만 수행되어야한다
- 단 하나의 커넥션 풀을 가지고 커넥션을 빌려 쓰기로 했으니 `static final` 선언 한다

# 자원 해제
- `try-with-resources` 구문 활용으로 대여 후 바로 반납하도록 하자

```java
public static void main(String[] args) {
	try {
		// 애플리케이션 실행
	} finally {
		Database.closeDataSource();
		System.out.println("DB closed");
	}
}
```
- 메인 메서드, 즉 애플리케이션이 완전 종료되는 시점에도 풀이 가진 모든 커넥션을 닫아주도록 하자

