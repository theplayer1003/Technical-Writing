
```java
public class TransactionSample {  
  
    private static final String DB_URL = "jdbc:h2:file:./data/practicedb_transaction";  
    private static final String USER = "sa";  
    private static final String PASS = "";  
  
    /**  
     * DB 세팅 -> 5000원 이체 시도 -> 성공 결과 출력 -> 없는 id에 이체 시도 -> 이체 실패 결과 출력  
     */  
    public static void main(String[] args) {  
        setupAccountDatabase();  
  
        System.out.println("\n-- 이체 전 잔액 --");  
        printBalances();  
  
        System.out.println("\n-- [task1] 5000원 이체 시도 (success case) --");  
        transferMoney(1, 2, 5000);  
  
        System.out.println("\n-- [task1] 5000원 이체 후 잔액 --");  
        printBalances();  
  
        System.out.println("\n-- [task2] 3000원 이체 시도 (fail case) --");  
        transferMoney(1, 99, 3000);  
  
        System.out.println("\n-- [task2] 3000원 이체 실패 후 잔액 --");  
        printBalances();  
    }  
  
    private static void transferMoney(int fromId, int toId, int amount) {  
        String withdrawSql = "UPDATE ACCOUNTS SET balance = balance - ? WHERE id = ?";  
        String depositSql = "UPDATE ACCOUNTS SET balance = balance + ? WHERE id = ?";  
  
        Connection conn = null;  
        PreparedStatement pstmtWithdraw = null;  
        PreparedStatement pstmtDeposit = null;  
  
        try {  
            conn = DriverManager.getConnection(DB_URL, USER, PASS);  
  
            conn.setAutoCommit(false);  
            System.out.println("-- 트랜잭션 시작 AutoCommit=false --");  
  
            pstmtWithdraw = conn.prepareStatement(withdrawSql);  
            pstmtWithdraw.setInt(1, amount);  
            pstmtWithdraw.setInt(2, fromId);  
            int withdrawResult = pstmtWithdraw.executeUpdate();
  
            pstmtDeposit = conn.prepareStatement(depositSql);  
            pstmtDeposit.setInt(1, amount);  
            pstmtDeposit.setInt(2, toId);  
            int depositResult = pstmtDeposit.executeUpdate();  
  
            if (withdrawResult == 1 && depositResult == 1) {  
                conn.commit();  
                System.out.println("-- 트랜잭션 커밋 완료, 이체 성공 --");  
            } else {  
                conn.rollback();  
                System.out.println("-- 트랜잭션 실패, 롤백 완료 --");  
            }  
  
        } catch (SQLException e) {  
            System.err.println("-- 트랜잭션 SQL 예외 발생, 롤백 실행 --");  
            try {  
                if (conn != null) {  
                    conn.rollback();  
                }  
            } catch (SQLException e2) {  
                e2.printStackTrace();  
            }  
            e.printStackTrace();  
        } finally {  
            try {  
                if (pstmtWithdraw != null) {  
                    pstmtWithdraw.close();  
                }  
                if (pstmtDeposit != null) {  
                    pstmtDeposit.close();  
                }  
                if (conn != null) {  
                    conn.close();  
                }  
            } catch (SQLException e) {  
                e.printStackTrace();  
            }  
        }  
    }  
  
    private static void printBalances() {  
        String sql = "SELECT * FROM ACCOUNTS";  
        try (Connection conn = DriverManager.getConnection(DB_URL, USER, PASS);  
             Statement stmt = conn.createStatement();  
             ResultSet rs = stmt.executeQuery(sql)) {  
  
            while (rs.next()) {  
                System.out.printf("ID : %d, Name : %s, Balance : %d\n",  
                        rs.getInt("id"),  
                        rs.getString("name"),  
                        rs.getInt("balance"));  
            }  
        } catch (SQLException e) {  
            e.printStackTrace();  
        }  
    }  
  
    private static void setupAccountDatabase() {  
        try (Connection conn = DriverManager.getConnection(DB_URL, USER, PASS);  
             Statement stmt = conn.createStatement()) {  
  
            stmt.executeUpdate("DROP TABLE IF EXISTS ACCOUNTS");  
            stmt.executeUpdate("CREATE TABLE ACCOUNTS (" +  
                    "id INT PRIMARY KEY, " +  
                    "name VARCHAR(255) NOT NULL, " +  
                    "balance INT NOT NULL)");  
  
            stmt.executeUpdate("INSERT INTO ACCOUNTS (id, name, balance) VALUES (1, 'pobi', 20000)");  
            stmt.executeUpdate("INSERT INTO ACCOUNTS (id, name, balance) VALUES (2, 'james', 10000)");  
  
            System.out.println("-- ACCOUNTS 테이블 설정 완료 --");  
        } catch (SQLException e) {  
            e.printStackTrace();  
        }  
    }  
}
```

- 지금까지의 학습을 통해 각각 한 건의 쿼리로 기초적인 DB 조작이 가능해졌다
- 하지만 두 건 이상의 쪼갤 수 없는, 논리적으로 하나인 쿼리가 있다면 이야기가 달라진다
- "A가 B에게 10,000원을 이체한다" 라는 시나리오는 두 가지 작업으로 분리된다
	- A의 잔고에서 10,000원을 차감한다 (UPDATE)
	- B의 잔고에 10,000원을 더한다 (UPDATE)
- 이를 분리해서 다룬다면 만일 두번째 절차에서 예상치 못한 오류로 프로그램이 멈췄을 때 A의 잔고만 돈이 줄어드는 결과가 된다
- 이를 해결하는 방법이 **트랜잭션** 이다
	- 트랜잭션이란, 더 이상 쪼갤 수 없는 하나의 논리적인 작업 단위

- `Auto-Commit` 모드는 실행 즉시 DB에 반영되는 상태를 말한다
	- `pstmt.executeUpdate()` -> 실행 즉시 바로 DB 반영
- 단 건의 쿼리에선 문제 없지만 앞서 말한대로 논리적으로 하나의 작업인 쿼리는 트랜잭션으로 처리한다
- 하나의 작업 단위이기 때문에 이는 반드시 모든 쿼리가 성공(`Commit`) 하거나, 단 하나라도 실패하면 전부 실패(`Rollback`) 시켜야 한다
- JDBC 에서의 구현
	- `conn.setAutoCommit(false)`
	  `commit` 코드를 만나기 전까지 DB 에 영구 반영 하지 않는다
	- ( 작업 수행 중.. )
	- `conn.commit();`
	  작업에 성공했다면 변경 사항을 DB 에 영구 반영하라
	- `conn.rollback();`
	  만약 중간에 예외가 발생해서 catch 되면 모든 변경 사항을 취소하고 트랜잭션을 원래대로 돌려라

## ACID

### Atomicity, 원자성
- 트랜잭션은 전부 실행되거나 전부 실행되지 않아야 한다
- 예시의 계좌 이체 처럼 출입금이 동시에 이루어지는 작업은 쪼개질 수 없는 하나의 단위여야한다

### Consistency, 일관성
- 트랜잭션이 성공적으로 완료되면, DB 는 항상 일관된 상태를 유지해야 한다
- 계좌 이체 전,후 간의 은행 시스템 총 잔액이 유지되어야 한다
- 그렇다면 일관성 체크를 위해 어떻게 해야하는가?
	- DBMS 제약 조건
	`NOT NULl` : Null 방지
	`UNIQE` : 중복 불가능
	`FOREIGN KEY` : 존재하지 않는 값을 참조 불가능
	`CHECK` : 가장 적극적인 일관성 보장
		`ALTER TABLE ACCOUNTS ADD CONSTRAINT chk_balance CHECK (balance >= 0);`
		위 DDL을 통해 잔액을 0원 이하로 만드는 작업은 DBMS가 즉시 거부하게 된다
		예외를 발생시키고 `catch` 블록에서 `rollback()` 함수가 호출되어 일관성이 지켜진다
	
	- 애플리케이션 비즈니스 로직
	`if (pobi.getBalance() < amount) { throw ... }`
	애플리케이션 쪽에서도 방어를 구현한다

### Isolation, 고립성, 격리성
- 한 트랜잭션이 실행 되는 동안 다른 트랜잭션은 중간에 간섭할 수 없어야 한다
- 예를 들어 A 계좌에 돈을 빼고 B 계좌에 돈을 입금한다고 가정하자
- A 계좌에 돈이 빠진 순간에 다른 트랜잭션이 A 계좌를 읽을 수 있으면 안된다
- 만약 B 계좌에 돈을 넣는 과정에서 문제가 생겼다면 원자성에 의해 이 작업은 롤백되어야 한다
- 다른 트랜잭션이 사이에 끼어들면 잘못된 정보를 읽게 된다
- 트랜잭션 끼리는 항상 작업 전, 후에만 접근이 가능해야 한다
- 이를 구현하기 위해 `Locking`, `MVCC` 등이 활용된다

### Durability, 지속성
- 트랜잭션이 성공적으로  `Commit` 되면 결과는 시스템 장애 등이 발생해도 영구적으로 저장되어야 한다
- 이는 애플리케이션 쪽 특성이 아닌 DBMS 에게 요구되는 특성이다
- `commit` DB 영구 반영 처럼 표현했지만 엄밀히는 실제 데이터 파일이 먼저 수정되는 것은 아니다
- 대신 작업 내역을 로그 파일에 먼저 기록해둔다
- 이 로그가 안전하게 작성되면 DBMS는 `commit` 성공이라고 응답한다
- 만약 이 이후에 장애가 발생하면 DBMS 는 이 로그를 바탕으로 데이터를 복구해 지속성을 충족시킨다








[[Mk2/2025woowa/precourse/week5/테크니컬_라이팅/DB_Study/기초/H2 데이터베이스를 사용해보자]]
