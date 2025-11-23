# SQL Injection의 원리와 방어 메커니즘
## 개요
SQL Injection 은 신뢰할 수 없는 외부 입력값이 데이터베이스 쿼리의 구조를 변조해 개발자가 의도하지 않은 명령을 실행하도록 하는 공격이다.

데이터베이스 입장에서 볼때 SQL 문은 명령어와 데이터가 섞여있는 문자열이다.
SQL Injection 의 핵심은 공격자가 입력한 데이터가 SQL 파서에 의해 명령어로 오인되어 해석, 실행되는 것이다.

## 공격 시나리오
1. `Statement` 취약점
`Statement` 인터페이스는 애플리케이션 레벨에서 문자열 결합 연산자(`+`) 를 통해 완성된 최종 SQL 문자열을 데이터베이스로 전송한다.
```java
String input = "' OR '1'='1"; // 공격 값
String sql = "SELECT * FROM USERS WHERE name = '" + input + "'";

// 합성된 sql 문자열: SELECT * FROM USERS WHERE name = ' OR '1'='1'
```

2. 데이터베이스 내부 처리 과정
	1. 최종 전송 받은 SQL 문자열은
	   `SELECT * FROM USERS WHERE name = ' OR '1'='1'` 이다
	2. Parsing: 구문 분석으로 SQL 문자열을 토큰 단위로 분해하여 문법적 오류를 검사한다
	3. Optimization & Compilation: 최적화 및 컴파일 단계로 실행 계획을 수립한다

이 과정에서 데이터베이스는 어느 부분이 개발자가 작성한 코드이고 사용자 입력값인지 관심이 없다. 입력값인 `' OR '1'='` 이 쿼리의 논리를 변경해버린다. `'1'='1'` 는 항상 참이기 때문에 WHERE 절은 항상 참 이 되어 모든 데이터를 반환하게 된다.

## 방어 시나리오
SQL Injection 을 막는 방법은 쿼리의 실행 구조(Code) 와 입력 데이터(Data) 를 철저히 분리하는 것이다. JDBC의 `PreparedStatement` 인터페이스가 이를 위해 **파라미터화된 쿼리** 방식을 사용한다.

1. `PreparedStatement` 동작 원리
	`PreparedStatement` 는 `Statement` 와 데이터베이스 통신 프로토콜이 근본적으로 다르다.
	1. 준비 및 컴파일
	2. 바인딩
	3. 실행
	
	애플리케이션은 먼저 쿼리의 틀만 데이터베이스에 전송해 컴파일한다. 변수 부분은 `?` (Placeholder) 와일드카드로 표시해둔다.
	이 시점에 쿼리가 파싱되고 컴파일되었기 때문에 **실행 게획이 고정된다.** 이후로는 쿼리의 문법 구조가 변할 수가 없게 된다.
	따라서, 공격 값인 `' OR '1'='1` 은 SQL 명령어의 일부로 해석되지 않고 **문자열 리터럴로 취급** 된다. 데이터베이스는 이 값을 통해 name 컬럼을 검색하려고 하고, 당연히 이런 값은 존재하지 않기 때문에 공격은 실패한다.
```java
String sql = "SELECT * FROM USERS WHERE name = ?"; // 쿼리 구조 정의

try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
    // 입력값을 구조와 분리된 '데이터'로 바인딩
    pstmt.setString(1, "' OR '1'='1");
    
    try (ResultSet rs = pstmt.executeQuery()) {
        // ... 결과 처리
    }
}
```
