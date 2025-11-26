# RDBMS 다대다 관계 해소

## 문제 정의
RDBMS 는 데이터를 행과 열로 이루어진 2차원 표로 표현한다. 이러한 태생적 한계 때문에 **다대다, N:M** 관계를 직접 표현하는데 한계가 있다.

```sql
CREATE TABLE Student (
	student_id INT PRIMARY KEY,
	student_name VARCHAR(50),
	course_id INT,
	FOREIGN KEY (course_id) REFERENCES Course(course_id)
);
```

위와 같은 테이블이 있고 `Course` 테이블을 참조한다고 가정해보자.
학생은 여러 과목을 수강할 수 있고, 과목은 여러 학생들에게 등록될 수 있다.

| student_id(PK) | student_name | course_id(FK) |                  |
| -------------- | ------------ | ------------- | ---------------- |
| 1              | pobi         | 100           | 저장 성공            |
| 1              | pobi         | 101           | **저장 실패, PK 중복** |

`Student` 와 `Course` 의 관계를 표현하려고 했지만 학생 아이디라는 기본 키가 중복 되어 여러 과목 수강을 표현할 수 없다.

| student_id | student_name | course_id |
| ---------- | ------------ | --------- |
| 1          | pobi         | 100       |
| 1          | pobi         | 101       |
| 1          | pobi         | 102       |

만약 PK 제약을 해제한다고 해도 같은 데이터가 불필요하게 중복 저장된다. 이는 저장 공간 낭비이며 학생 이름이 변경될 때 모든 행을 수정해야 하는 갱신 이상을 초래한다.

즉, **N:M 관계** 는 2차원 표인 RDBMS 에서 직접적으로 표현하기 힘들다.

## 해결 방법

### 연결 테이블
다대다 관계를 해소하는 제일 표준적인 방법이다. 연결 테이블이란 다대다 관계가 형성되는 두 테이블 사이에 연결 테이블을 만들어 1:N 관계를 거쳐가도록 표현하는 테이블을 말한다.

- 구조: `Student` ◀(1:N)─ `Enrollment` ─(N:1)▶ `Course`
- 역할: `Enrollment` 수강 테이블을 만들어 둘의 관계를 표현한다.

```sql
CREATE TABLE Student (
	student_id INT PRIMARY KEY AUTO_INCREMENT,
	student_name VARCHAR(50) NOT NULL
);

CREATE TABLE Course (
	course_id INT PRIMARY KEY AUTO_INCREMENT,
	course_name VARCHAR(100) NOT NULL UNIQUE
);
```

```sql
CREATE TABLE Enrollment (
	student_id INT,
	course_id INT,
	
	-- 기본 키를 복합 키로 설정해 (학생+과목) 조합의 중복을 방지할 수 있다
	PRIMARY KEY (student_id, course_id),
	
	CONSTRAINT fk_enroll_student
		FOREIGN KEY (student_id) REFERENCES Student (student_id)
		ON DELETE CASCADE,
		
	CONSTRAINT fk_enroll_course
		FOREIGN KEY (course_id) REFERENCES Course (course_id)
		ON DELETE RESTRICT
);
```

등록 테이블을 만들어 두 테이블을 연결한다. 이제 등록 테이블에 어떤 학생이 어떤 과목을 수강하는지 표현할 수 있다.

| student_id | student_name |
| ---------- | ------------ |
| 1          | pobi         |
| 2          | James        |

| course_id | course_name |
| --------- | ----------- |
| 100       | JAVA        |
| 101       | DB          |
| 102       | OpenMission |

| student_id | course_id |
| ---------- | --------- |
| 1          | 100       |
| 1          | 101       |
| 1          | 102       |
| 2          | 101       |

### 최신 RDBMS의 JSON/Array 타입
PostgreSQL, MySQL(5.7+) 등 최신 RDBMS 에서 JSON 이나 Array 타입을 지원한다. 이를 활용해 연결 테이블 없이 구현이 **가능은 하다**.

스키마가 단순해지고, 조회 시 `JOIN` 없이 한 번에 가져 올 수 있어 편리하지만, 무결성이 위배되며 데이터 중복 발생, 검색 성능이 떨어지는 등 단점이 있다.