```java

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
```

- 왜 앞선 예제에선 `try-with-resources` 구문으로 깔끔하게 자원을 해제했는데 트랜잭션 코드에서는 사용하지 못하는가?
- `try resources` 구문의 스코프에 대해 알아야 한다
	- `try ( ... ) { ... }`
	- ( ... ) 안의 자원은 뒤에 따라오는 { ... } 안에서만 유효하다
	- 즉, try 블록이 정상 종료되거나 예외가 발생해 catch 블록으로 넘어가거나, 두 케이스 모두 상관 없이 try 를 벗어나 resource.close() 가 자동 호출된다
- 코드에서 보이는대로 `conn.rollback()` 은  `catch` 에서도 일어난다
- `try resources` 에서는 이미 `conn` 이 자원 해제되어 예외가 발생한다

- 이러한 이유로 try 문 바깥에 자원들을 먼저 선언해 스코프가 try, catch, finally 구문 모두에 닿도록 해야한다


# 아니 너무 복잡하지 않아요?
- 그렇다. 너무 복잡하다
- 이 복잡한 코드를 해결하기 위해 `Spring` 이라는 프레임워크를 사용한다
- 하지만 이번 미션은 순수 자바 사용이 목표 중 하나다
- 이번 미션을 통해 불편함을 깨닫고 나아중에 스프링의 위대함을 느껴보도록 하자