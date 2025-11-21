# 트랜잭션과 ACID

## 트랜잭션의 정의

### 개념 및 정의
트랜잭션은 데이터베이스의 상태를 변화시키는 하나의 논리적 작업 단위를 의미한다. 단건의 SQL 문을 실행시키는 것이 아닌, 여러 개의 작업을 묶어서 **'쪼개지면 안되는 하나의 덩어리'** 로 처리하는게 트랜잭션이다.

### 예시 시나리오
계좌 이체를 한다고 가정해보자.
'A 가 B 에게 10,000원 을 이체한다' 는 작업이 필요하다고 할때, 실제 데이터베이스 입장에서는 두 가지 작업이 된다.
1. A 의 잔고에서 10,000 원을 차감한다. (`UPDATE`)
2. B 의 잔고에 10,000 원을 추가한다 (`UPDATE`)

이 두 작업은 논리적으로 하나의 작업과 같다. 만약 1번만 수행되고 2번이 수행 되지 않는다면 치명적인 데이터 불일치가 발생한다.
이것이 트랜잭션의 필요성이며 **'둘 다 성공하거나, 아니면 둘 다 실패하거나'** 가 보장되어야하는 이유다.

## ACID 원칙
ACID 는 데이터베이스가 트랜잭션의 안전한 처리를 보장하기 위한 4가지 성질이다.

### Atomicity, 원자성
- 정의: 트랜잭션 내의 모든 작업은 전부 성공하거나 전부 실패해야 한다.
- 보장 방법: 작업 중 하나라도 실패하면 이전의 성공한 작업도 모두 취소하는 롤백(Rollback) 을 수행한다.

### Consistency, 일관성
- 정의: 트랜잭션이 성공적으로 완료되면, 데이터베이스는 항상 일관된 상태를 유지해야 한다.
- 보장 방법:
	- DBMS의 제약 조건: `NOT NULL`, `UNIQUE`, `FOREIGN KEY`, `CHECK` 등의 제약 조건을 정의하고 이를 위반하면 바로 롤백 시킨다.
	- 비즈니스 로직: Application 쪽에서도 잔고가 마이너스가 될 수 없다는 규칙 등을 검증한다.

### Isolation, 고립성, 격리성
- 정의: 동시에 실행되는 트랜잭션들끼리 서로 영향을 주지 않고 독립적으로 수행되어야 한다.
- 문제 시나리오:
	1. \[Tx1] A 의 잔고에서 10,000 원을 차감한다.
	2. \[Tx2] B 의 잔고에 돈을 추가하기 전에 다른 트랜잭션이 A 의 잔고를 조회한다.
	3. \[Tx1] B 의 잔고에 10,000 원을 추가하는 과정에서 장애가 발생해 롤백이 일어난다.
	4. \[Tx1] A 의 잔고는 다시 10,000 원이 추가된다.
	5. \[Tx2] 다른 트랜잭션은 잘못된 A 의 잔고를 읽고 작업을 처리한다.
- 보장 방법: 데이터베이스가 `Locking`, `MVCC` 등의 기술을 활용해 트랜잭션 간의 간섭을 제어한다.

### Durability, 지속성
- 정의: 성공적으로 완료(커밋) 한 결과는 시스템 장애 등이 발생해도 영구적으로 저장되어야 한다.
- 보장 방법: 애플리케이션 레벨이 아닌 DBMS 에 요구되는 특성으로 데이터를 실제 디스크 파일에 쓰기 전에, 작업 내용을 로그 파일에 먼저 기록하고 장애 대응 이후 로그를 읽어 복구한다.

## JDBC 에서 트랜잭션 구현

### 트랜잭션 프로세스
1. `conn.setAutoCommit(false)`: 트랜잭션의 시작
2. `conn.prepareStatement(sql)`: 동적 SQL 실행 객체 생성
3. `PreparedStatement.executeUpdate()`: 비즈니스 로직(SQL) 수행, 아직 DB 미반영 상태
4. `conn.commit()`: 모든 작업 성공 시, DB에 영구 반영
5. `conn.setAutoCommit(true)`: 트랜잭션 모드 종료
6. `conn.close()`: 자원 반환

### 실제 코드 예시

```java
Connection conn = null;  
PreparedStatement pstmtWithdraw = null;  
PreparedStatement pstmtDeposit = null;  
  
try {  
    conn = DriverManager.getConnection(DB_URL, USER, PASS);  
  
	// [1] 트랜잭션 시작, 자동 커밋 해제  
    conn.setAutoCommit(false);  
  
	// [2,3] 동적 SQL 실행 객체 생성 및 비즈니스 로직 수행
    pstmtWithdraw = conn.prepareStatement(withdrawSql);  
    pstmtWithdraw.setInt(1, amount);  
    pstmtWithdraw.setInt(2, fromId);  
    int withdrawResult = pstmtWithdraw.executeUpdate();    
  
    pstmtDeposit = conn.prepareStatement(depositSql);  
    pstmtDeposit.setInt(1, amount);  
    pstmtDeposit.setInt(2, toId);  
    int depositResult = pstmtDeposit.executeUpdate();  
  
	// [4] 모든 작업이 성공했다면 DB에 commit
    if (withdrawResult == 1 && depositResult == 1) {  
        conn.commit();  
        System.out.println("-- 트랜잭션 커밋 완료, 이체 성공 --");  
    } else {  
        conn.rollback();  
        System.out.println("-- 트랜잭션 실패, 롤백 완료 --");  
    }  
  
} catch (SQLException e) {  
	// [5] 예외 발생 시 롤백 수행 코드
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
	// [6] 리소스 정리 및 AutoCommit 복원  
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
```

- `conn.setAutoCommit(false)`
	- `Auto-Commit` 모드는 실행 즉시 DB에 반영되는 상태를 말한다.
	- 하나의 트랜잭션은 여러 쿼리로 작성되기 때문에 이 모드를 끄는 것으로 시작한다.
- `PreparedStatement.executeUpdate()`
	- 쿼리 실행 결과로 변경된 데이터 행의 개수를 반환한다
- `conn.commit()`
	- 작업에 성공했다면 이 메서드를 호출해 변경 사항을 DB 에 영구 반영한다
- `conn.rollback()`
	- 만약 작업에 문제가 발생했다면 반드시 롤백 메서드를 호출 시켜 트랜잭션 블록 안의 모든 작업이 취소되도록 한다