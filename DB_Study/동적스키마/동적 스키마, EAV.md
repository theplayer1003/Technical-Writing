# 정의
- 동적 스키마 란 `ALTER TABLE` 같은 DDL 을 사용해 스키마 구조를 물리적으로 변경하지 않고도, 애플리케이션 레벨에서 속성을 자유롭게 추가, 삭제, 변경 할 수 있도록하는 요구사항, 개념을 말한다
- Entity-Attribute-Value 란 동적 스키마 요구사항을 RDBMS 에서 구현하기 위한 여러 설계 패턴 중 하나이다

# 동적 스키마
- 지금까지의 학습은 정적 스키마로 무결성, 일관성 등에 대해 뛰어나고 안정적이지만 스키마가 경직되어 있다는 한계가 있다

## 스키마가 경직되어 있다?
- 쉽게 말해 설정한 표에서 벗어나질 못한다는 것
- 예를 들면 우리가 작성한 `Student` 테이블에는 `student_id`, `student_name` 이 있을 수 있다
- 이후 요구사항의 확장으로 `hobby`, `MBTI` 등 다양한 속성이 필요로 해진다고 가정하자
- 정적 스키마 는 새롭게 테이블을 변경할 필요가 있다. `ALTER TABLE`
- 만약 테이블에 이미 무수한 데이터가 저장되어 있다면?
- 들어 있는 데이터 만큼 비용이 늘어나게 된다
- 비용을 지불하고 바꿨다고 가정하자
- `hobby` 가 없는 학생들이 있을 수도 있다
- 무수한 학생들 중에 소수만이 `hobby` 를 가지고 있다면 나머지는 `NULL`  값으로 공간과 성능이 크게 낭비된다

## 이를 해결하고자 하는게 EAV
- 기존 정적 스키마의 경직이 어떻게 문제가 되는지 알아봤다. 이제 `EAV` 로 이를 해결해보자
- `EAV` 는 `속성` 을 `열` 로 정의하지 않고 `행` 으로 저장하는 아이디어다

- 정적 스키마의 경우

| user_id | username | email           | **hobby** | **mbti** |
| :------ | :------- | :-------------- | :-------- | :------- |
| 1       | Pobi     | pobi@woowa.com  | Bowling   | ENFP     |
| 2       | James    | james@woowa.com | Coding    | `NULL`   |

- 동적 스키마의 경우 (`EAV` 의 경우)

| entity_id | attribute_name | value             |     |
| :-------- | :------------- | :---------------- | --- |
| 1         | 'username'     | 'pobi'            |     |
| 1         | 'email'        | 'pobi@woowa.com'  |     |
| 1         | 'hobby'        | 'Bowling'         |     |
| 1         | 'mbti'         | 'ENFP'            |     |
| 2         | 'username'     | 'James'           |     |
| 2         | 'email'        | 'james@woowa.com' |     |
| 2         | 'hobby'        | 'Coding'          |     |

- 정적 스키마는 변경사항이 가로로 확장된다
- 동적 스키마는 변경사항이 세로로 확장된다
- 정적 스키마는 값이 없을 경우 `NULL` 이지만 동적 스키마는 그냥 행 자체가 존재하지 않는다

## 코드로 작성해보자

- Entity Table
	- 정적 스키마와 유사한 느낌이다
	- 데이터가 존재하기 위해 필수적인 값을 정의한다
	- `student_name` 은 필수 값은 아니지만 사실상 모든 학생이 가져야만 하는 값이다
	- `순수 EAV` 모델에서는 정말로 필수적인 값 `student_id` 만 Entity 로 가지지만 이처럼 사실상 무조건 따라오는 값인 이름도 Entity 에 선언하는 것을 '하이브리드 모델' 이라고 한다
	- 이를 통해 큰 성능 향상을 얻을 수 있다
```sql
CREATE TABLE Student (
	student_id INT PRIMARY KEY AUTO_INCREMENT,
	student_name VARCHAR(100) NOT NULL
);
```

- Attribute Table
	- 엔티티가 가질 속성들을 정의하는 테이블이다
	- 학생은 취미를 가진다 -> 취미가 엔티티가 가질 속성
	- EAV 에서 속성은 다양한 값을 저장해야하므로 모두 문자로 취급된다
	- 정적 스키마에서는 각 속성에 어울리는 자료구조를 가질 수 있지만 EAV 에서는 취미인 '볼링' 도, 생일인 '941003' 도 모두 글자로 취급한다
```sql
CREATE TABLE AttributeDefinition (
	attr_id INT PRIMARY KEY AUTO_INCREMENT,
	attr_name VARCHAR(50) NOT NULL UNIQUE
)
```

- Value Table
	- 실제 값을 저장하는 테이블
	- Entity Table 과 Attribute Table 을 연결하는 N:M 연결 테이블이 된다
```sql
CREATE TABLE StudentAttributeValue (
	student_id INT,
	attr_id INT,
	
	value TEXT,
	
	PRIMARY KEY (student_id, attr_id),
	FOREIGN KEY (student_id) REFERENCES Student (student_id) ON DELETE CASCADE,
	FOREIGN KEY (attr_id) REFERENCES AttributeDefinition (attr_id) ON DELETE RESTRICT
)
```


- 이제 우리는 새로운 속성이 추가되어도 `ADD COLUMN` 이 아닌 Attribute 테이블과 Value Table 에 각각 속성과 값을 추가해주기만 하면 된다
- DDL 작업이 DML 작업으로 바뀐 것, 이것이 동적 스키마가 구현된 모습이다



## 단점은?
- 앞서 언급한대로 타입 안정성을 잃는다
- 데이터베이스가 가지는 타입 안정성을 잃기 때문에 모든 책임 검증이 애플리케이션에서 이루어져야 한다
	- `CHECK (age > 0)` 불가능, application 쪽에서 구현해줘야 한다
- 쿼리가 복잡해진다
- 행으로 저장된 데이터를 열 처럼 보이게 가져와야하는데 이를 위해 많은 `JOIN` 과 `SubQuery` 가 필요해진다. 이를 `PIVOT` 작업이라고 한다
```sql
SELECT
    s.student_name,
    (SELECT v.value FROM StudentAttributeValue v JOIN AttributeDefinition a ON v.attr_id = a.attr_id 
     WHERE v.student_id = s.student_id AND a.attr_name = 'hobby') AS hobby,
    (SELECT v.value FROM StudentAttributeValue v JOIN AttributeDefinition a ON v.attr_id = a.attr_id 
     WHERE v.student_id = s.student_id AND a.attr_name = 'mbti') AS mbti
FROM Student s
WHERE s.student_id = 1;
```

