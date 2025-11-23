# JDBC 핵심 인터페이스 명세

## 개요
JDBC(Java Database Connectivity) 표준 API(java.sql) 의 핵심 인터페이스 `Connection`, `PreparedStatement`, `ResultSet` 의 역할과 메서드 명세를 다룬다.

## Connection
데이터베이스와의 물리적 연결 세션을 담당하는 객체다. 트랜잭션 관리와 쿼리 실행 객체 생성을 담당한다.

### 주요 속성 및 메서드
| **메서드 시그니처**                                     | **설명**                                                | **비고**                                           |
| ------------------------------------------------ | ----------------------------------------------------- | ------------------------------------------------ |
| `Statement createStatement()`                    | 파라미터가 없는 정적 SQL 실행을 위한 `Statement` 객체를 반환한다.          |                                                  |
| `PreparedStatement prepareStatement(String sql)` | 파라미터가 포함된 동적 SQL 실행을 위한 `PreparedStatement` 객체를 반환한다. | **실무에서는 보안(SQL Injection) 상의 이유로 이것만 사용하길 권장한다** |
| `void setAutoCommit(boolean autoCommit)`         | 트랜잭션 자동 커밋 모드를 설정한다. (기본값: `true`)                    | 트랜잭션 시작 시 `false` 설정                             |
| `void commit()`                                  | 현재 트랜잭션의 모든 변경 사항을 DB에 영구 반영한다.                       |                                                  |
| `void rollback()`                                | 현재 트랜잭션의 모든 변경 사항을 취소하고 이전 상태로 되돌린다.                  |                                                  |
| `void close()`                                   | 데이터베이스 연결을 종료하거나, 커넥션 풀에 반납한다.                        | `AutoCloseable` 구현                               |

## PreparedStatement
동적 SQL 쿼리를 실행하는 객체다. `Statement` 를 상속받았으며, Pre-compile 기능을 통해 SQL 재사용이 가능하고 파라미터 바인딩을 통해 SQL Injection 공격을 방지할 수 있다.

### 주요 속성 및 메서드
| **메서드 시그니처**                                   | **설명**                                                          | **비고**          |
| ---------------------------------------------- | --------------------------------------------------------------- | --------------- |
| `void setString(int parameterIndex, String x)` | 지정된 순서(`parameterIndex`)의 파라미터(`?`)에 문자열 값을 바인딩한다.              | 인덱스는 **1부터 시작** |
| `void setInt(int parameterIndex, int x)`       | 지정된 순서의 파라미터에 정수 값을 바인딩한다.                                      |                 |
| `ResultSet executeQuery()`                     | `SELECT` 구문을 실행하고 결과 집합(`ResultSet`)을 반환한다.                     |                 |
| `int executeUpdate()`                          | `INSERT`, `UPDATE`, `DELETE` 구문을 실행하고, 영향을 받은 행(Row)의 개수를 반환한다. |                 |

## ResultSet
`SELECT` 쿼리의 실행 결과를 추상화한 객체다. 내부에 커서가 구현되어 있어 데이터에 접근할 수 있다. `while(rs.next())` 를 통해 순회하며 데이터에 접근한다.

### 주요 속성 및 메서드
| **메서드 시그니처**                    | **반환 타입** | **설명**                                                |
| ------------------------------- | --------- | ----------------------------------------------------- |
| `next()`                        | `boolean` | 커서를 다음 행으로 이동시킨다. 데이터가 있으면 `true`, 없으면 `false`를 반환한다. |
| `getString(String columnLabel)` | `String`  | 현재 커서 위치의 행에서 컬럼 이름에 해당하는 값을 문자열로 가져온다.               |
| `getInt(String columnLabel)`    | `int`     | 현재 커서 위치의 행에서 컬럼 이름에 해당하는 값을 정수로 가져온다.                |
| `getObject(String columnLabel)` | `Object`  | 범용적인 객체 타입으로 값을 가져온다.                                 |

## Statement
정적 SQL 쿼리를 실행하는 객체다. **보안이나 성능 문제로 실무에서는 사용하지 않는다.**