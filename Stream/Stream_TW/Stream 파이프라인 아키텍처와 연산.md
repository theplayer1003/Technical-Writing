# Stream 파이프라인 아키텍처와 연산

# 개요
- Stream API 는 데이터 처리 연산을 위해 반드시 3단계의 구조를 거친다.

1. `Source`: 데이터의 원천인 소스
2. `Intermediate Operations`: 중간 연산
3. `Terminal Operation`: 종단 연산

## Lazy Evaluation

```java
names.stream().filter(...).map(...)...
```

Stream 코드를 처음 접하면 메서드가 호출되는 순서대로 즉시 실행될 것(`Eager Evaluation`) 이라는 착각을 하게 쉽다. (filter -> map -> ...)

```java
List<String> names = Arrays.asList("Java", "Kotlin", "Scala");

Stream<String> stream = names.stream()
	.filter(name -> {
		System.out.println("Filtering: " + name);
		return name.length() > 3;
	})
	.map(name -> {
		System.out.println("Mapping: " + name);
		return name.toUpperCase();
	});
	
System.out.println("파이프라인 지연 로딩 테스트");
```

```text
파이프라인 지연 로딩 테스트
```

하지만 실제로는 **종단 연산**이 호출되기 전까지 Stream 은 아무런 작업도 수행하지 않는다.

```java
List<String> names = Arrays.asList("Java", "Kotlin", "Scala");

Stream<String> stream = names.stream()
	.filter(name -> {
		System.out.println("Filtering: " + name);
		return name.length() > 3;
	})
	.map(name -> {
		System.out.println("Mapping: " + name);
		return name.toUpperCase();
	});
	
System.out.println("파이프라인 생성 완료");

stream.forEach(s -> {});
```

```text
파이프라인 생성 완료
Filtering: Java
Mapping: Java
Filtering: Kotlin
Mapping: Kotlin
Filtering: Scala
Mapping: Scala
```

- `stream()`: 스트림의 시작점 Head 를 표시한다
- `filter(), map()`: 수행할 작업을 `Stage` 객체를 생성해 연결한다
- `forEach()`: 종단 연산이 호출되면 연결된 `Stage` 객체들을 역추적해 수행 계획을 파악하고 데이터 소스에게 데이터를 보내도록 요청해 작업을 실행한다

### 지연 연산이 필요한 이유
1. 루프 퓨전: `filter` 와 `map` 등 여러 단계를 하나의 반복문으로 통합해 최적화하기 위함
2. 쇼트 서킷: 불필요한 데이터를 처리하지 않고 즉시 종료하기 위함

## 핵심 연산

### 중간 연산, Intermediate Operations
- 무상태, Stateless
이전 데이터의 상태를 알 필요 없이 개별 요소를 독립적으로 처리한다. 메모리 부하가 적어 많은 양의 데이터 처리가 수월하다.

	- `filter(Predicate)`: 조건에 맞는 것만 통과 시키는 연산
	```java
	stream.filter(name -> name.length() > 3)
	// 길이가 3 초과인 문자열만 통과
	```

	- `map(Function)`: 데이터를 변환 시키는 연산
	```java
	stream.map(String::toUpperCase)
	// 문자열을 대문자로 변환
	```
	
	- `flatMap(Function)`: 요소를 스트림으로 변환한 뒤 하나로 평탄화하는 연산
	```java
	stream.flatMap(Collection::stream)
	// [[a, b], [c, d]] -> [a, b, c, d]
	```
	
	- `peek(Consumer)`: 데이터를 소비하지 않고 확인만 하는 연산, 디버깅용
	```java
	stream.peek(System.out::println)
	// 데이터 흐름을 출력만 하고 다음 단계로 넘어감
	```

- 상태 유지, Stateful
처리를 위해 이전 데이터나 전체 데이터를 버퍼에 저장할 필요가 있는 연산들이다. 버퍼 메모리, 연산을 위한 모든 데이터 순회 등 비용이 높다.

	- `sorted(Comparator)`: 데이터를 정렬하는 연산, 정렬을 하려면 한번 모든 데이터를 다 모아야하기 때문에 가장 비싼 연산이다
	```java
	stream.sorted(Comparator.reverseOrder())
	// 역순 정렬
	```
	
	- `distinct()`: 중복을 제거하는 연산, 중복 여부를 체크하려면 비교를 위해 상태를 저장해야만 한다
	```java
	stream.distinct()
	// [1, 1, 2] -> [1, 2]
	```
	
	- `limit(long)`: 개수를 제한하는 연산, 개수를 세기 위해 상태를 저장해야만 한다
	- `skip(long)`: 건너뛰기 연산
	```java
	stream.skip(5).limit(10)
	// 처음 5개는 버리고 남은 부분의 10개만 연산
	```

- 기타, 최적화
데이터 자체를 변환하진 않지만 파이프라인의 처리 방식을 변경해 성능에 영향을 준다.

	- `unoredered()`: 순서가 중요하지 않음을 명시해 병렬 처리를 최적화하는 기능
		- 주로 병렬 스트림에서 distinct(), limit(), groupingBy() 등 과 함께 쓰인다
		- 순서 제약을 제거하여 스레드 간 동기화 비용을 줄여 성능을 최적화한다
	```java
	stream.parallel().unordered().limit(5)
	// 병렬 처리 시 순서에 상관없이 5개만 빨리 가져옴
	```

### 종단 연산, Terminal Operations
- Reduction
연산한 데이터들로 하나의 결과를 만들어낸다.

	- `collect(Collector)`: List, Map, Set 등으로 결과 모아 만든다
	```java
	stream.collect(Collectors.toList())
	stream.collect(Collectors.joining(", "))
	// list로 반환, 문자열로 합치기
	```
	
	- `reduce(BinaryOperator)`: 누적 계산, 합계 구하기 같은 것들
	```java
	stream.reduce(0, Integer::sum)
	// 초기값을 0으로 하고 모든 숫자의 합계를 구함
	```
	
	- `count()`: 결과의 개수를 반환
	- `max(Comparator)`: 최대값
	- `min(Comparator)`: 최소값

- Short-circuiting
모든 요소를 순회하지 않아도 결과를 낼 수 있다. 무한 스트림 처리에 필수인 종단 연산이다.

	- `anyMatch(Predicate)`: 하나라도 만족시키면 즉시 true 반환 후 종료
	- `allMatch(Predicate)`: 모두 만족해야 true 반환, 하나라도 불만족하면 즉시 false 반환 후 종료
	- `noneMatch(Predicate)`: 모두 만족하지 않으면 true 반환, 하나라도 만족하면 즉시 false 반환 후 종료
	- `findFirst()`: 첫번째 요소 반환 (`Optional` 로 반환)
	- `findAny()`: 아무거나 하나 반환 (`Optional` 로 반환), 병렬 처리 시 `findFirst` 보다 빠르다.

- Side-effect
결과를 반환하는 것이 아니라 요소를 순회하며 외부 세계에 영향을 주는 작업을 수행한다.

	- 주의
	`forEach` 내부에서 리스트 값을 `add` 하는 등 외부 상태를 변경하는 로직은 지양해야 한다. 동시성 문제 발생 등의 문제로 출력 용도로만 사용하는 것이 권장된다.