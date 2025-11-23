# DB 연결 비용과 커넥션 풀

## 문제 상황
데이터베이스 프로그래밍에서 `DriverManager.getConnection()` 은 가장 비용이 높은 메서드이다. 애플리케이션이 데이터베이스와 물리적 연결을 맺는 과정으로 그 때마다 다음의 복잡한 통신 과정이 발생한다.

1. 네트워크 연결: TCP/IP 3-way Handshake를 통한 신뢰성 있는 연결을 수립한다.
2. 인증: DB 아이디와 비밀번호를 전송하고 DB 서버에서 이를 검증하여 접근 권한을 확인한다.
3. 리소스 할당: DB 서버는 해당 연결을 위한 세션 메모리 영역을 확보하고 관리 객체를 생성한다.

이 과정은 단순한 쿼리를 한번 수행하는 것 보다 수천 배 많은 리소스를 소모한다.
만약 사용자 요청마다 이 과정이 반복되면 응답 속도가 현저히 느려지게 된다.
또 트래픽이 급증하는 경우 DB 서버가 감당할 수 있는 동시 연결 수를 초과하여 서비스 장애가 발생할 수 있다.

## 해결책
Connection Pool 을 이용해 문제를 해결한다.

### 커넥션 풀
커넥션 풀은 애플리케이션 시작 시점에 필요한 만큼의 커넥션을 미리 생성해 풀에 보관하고 필요할때 꺼내 쓰고, 재사용하는 전략이다.

1. 초기화: 애플리케이션이 구동되고 미리 설정한 개수만큼 물리적 연결(커넥션)을 미리 맺어둔다.
2. 대여: 요청이 들어오면 풀에서 유후 상태인 커넥션을 빌려준다.
3. 사용: 빌린 커넥션으로 SQL 작업을 진행한다.
4. 반납: 사용이 끝나면 연결을 끊는게 아닌, 풀에 되돌려 놓는다.

이를 통해 최초의 연결 생성 비용을 제외하면 작업 때마다 연결 비용이 발생하지 않으며, 정해진 수의 연결만 유지하므로 DB 부하를 효율적으로 관리할 수 있다.

### HikariCP와 DataSource
Java 에서는 표준 인터페이스 `DataSource` 와 그 구현체 `HikariCP` 를 사용해 커넥션 풀을 구현한다.
`DataSource` 는 커넥션을 획득하는 방법을 추상화한 인터페이스로 개발자가 구체적으로 어떤 라이브러리에 대한 구현체인지 알 필요가 없도록 해준다.

- 설정 및 생성 코드
```java
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
```

- `DataSource.getConnection`
DataSource로부터 커넥션을 대여 한다.
- `config.setJdbcUrl`, `config.setUsername()`, `config.setPassword()`
필수 연결 정보를 설정한다.
- `setMaximumPoolSize()`, `setConncetionTimeout`, `setPoolName`
성능 튜닝 핵심 옵션을 설정한다.


>[!tip] `mamximumPoolSize`
>성능에 가장 영향을 미치는 옵션으로 많을수록 좋다고 생각할 수 있지만 너무 많은 커넥션은 CPU의 스레드 간 전환 시간이 늘어나 성능을 저하시킨다.
>
>`CPU Core Count * 2 + 유효 디스크 수` 공식이 권장 된다.
>일반 웹 애플리케이션 환경에서는 10~20 개 정도의 설정으로 충분히 트래픽 감당이 가능하다.

