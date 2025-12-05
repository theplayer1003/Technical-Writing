# 패러다임의 전환

## 외부 반복
외부 반복은 클라이언트, 개발자가 작성한 코드가 컬렉션의 순회를 명시적으로 제어하는 방식임.
예를 들어 우리가 자주 쓰는 `for-each` 가 그러함. 내부적으로 `Iterator` 를 사용해 반복하고 이것은 외부 반복임.

```java
List<String> names = Arrays.asList("Pobi", "Crong", "James");
for (String name : names) {
	sout(name);
}

// 컴파일 후 이렇게 바뀜
Iterator<String> iterator = names.iterator();
while (iterator.hsaNext()) {
	String name = iterator.next();
	sout(name);
}
```

핵심은 데이터를 가져오는 행위와 데이터를 처리하는 행위가 클라이언트 코드 내에 섞여 있다는 것.
`hasNext()`, `next()` 등을 우리(개발자)가 스스로 작성한거나 마찬가지임. 우리가 관리하고 조종함.

>좋은거 아닌가?

이것은 순차적 처리 순서가 코드에 고정(하드코딩)되어 있다는걸 의미함. 이런 경우 병렬화가 어려움.

>그게 무슨 소리야?

```java
long sum = 0;
for (int i = 0; i< list.size();i++) {
	sum += list.get(i);
}
```

이런 코드를 병렬화 하려면 어떻게 해야할까?
이 코드는 이미 순서가 정해져 있음. 0일 때 처리하고, 1일 때 처리하고, 2일 때 처리하고 ...
데이터 소스와 처리 로직이 강하게 결합되어 있다고 표현함.
결국 병렬화를 위해 새로 코드를 작성해야함

```java
final int mid = list.size() / 2;
long[] sums = new long[2];

Thread t1 = new Thread(() -> {
	for (int i = 0; i < mid; i++) {
		sums[0] += list.get(i);
	}
});

Thread t2 = ....

t1.start();
t2.start();

try {
	t1.join();
	t2.join();
} catch ~

long finalSum = sums[0] + sums[1];
```

직접 작업을 분할해서 정의하고 직접 스타트하고  직접 기다리고 직접 각 결과를 합쳐야함
동시성 이슈도 해결해야함

비즈니스 로직에 비해 병렬 처리화 하는 코드가 더 많아지는 일종의 주객전도 같은 일이 일어남





## 내부 반복
내부반복은 우리(개발자)가 반복을 하는게 아님. 개발자는 '이걸 반복 시켜줘' 라고 시키기만 함. 누구한테? `Stream` 라이브러리 한테.
방식이 명령형에서 선언형으로 바뀜. 마치 SQL 처럼 우리는 DBMS 내부에서 무슨 일이 일어나는지는 관심이 없고 '이거이거 찾아줘' 라고 말하기만 함.

코드의 '제어권'(위 예시에서 직접 next(), hasNext()를 다룬 것) 이 우리가 작성하는 코드에서 라이브러리 안으로 이동한 것.

단순히 위치가 이동한게 아니라 책임의 분리가 일어난 거임.
우리는 시키기만 할 것이고 어떻게 할지는 stream 라이브러리가 알아서 함.
병렬화를 위해 이렇게 저렇게 코드를 고쳐야 했다면 이제 '병렬화 해줘' 하면 알아서 잘 병렬화 해서 실행 해줌.

```java
List<String> names = Arrays.asList("Pobi", "Crong", "James");

names.stream().forEach(name -> System.out.println(name));
```

이제 개발자는 `for`, `while` 등의 반복 처리 로직에 대해 고민하지 않고 필터 등만 조합해서 해야할 일을 지시함. 함수 던져주고 이걸 반복하라고 시킴. 제어권이 stream 라이브러리한테 있으니까.

```java
long sum = list.stream()
			.parallel() // '병렬화 해'. 끝.
			.mapToLong(i -> i)
			.sum();
```

병렬화도 반복을 던져주면서 병렬화도 하라고 지시하기만 하면 됨.



# 함수형 인터페이스와 람다

## 함수형 인터페이스
이게 갑자기 왜 나옴?

이게 있어서 Stream 이 가능해짐
앞선 내용을 통해 내부 반복을 위해 stream 라이브러리에 행위를 전달해야한다는걸 알 수 있음

자바는 객체지향 언어이기 때문에 메서드 단독으로 존재할 수 없고 반드시 클래스, 객체에 속해야 함.
이 부분을 해결하기 위해 나온게 함수형 인터페이스와 람다임

함수형 인터페이스란 단 하나의 추상 메서드 SAM,Single Abstract Method 만을 가지는 인터페이스를 뜻함.
```java
@FuntionalInterface
public interface MyFunction {
	void run();
}
```

보이는대로 하나의 추상 메서드만 가지고 있음. 어노테이션으로 명시적으로 선언해줄 수 있음.


## Stream API 4대 핵심 인터페이스

| 인터페이스            | 추상메서드               | 역할                                | Stream 매핑 메서드                         |
| ---------------- | ------------------- | --------------------------------- | ------------------------------------- |
| `Predicate<T>`   | `boolean test(T t)` | `T` 를 받아 `boolean` 반환(판별)         | `filter(Predicate)`                   |
| `Function<T, R>` | `R apply(T t)`      | `T` 를 받아 `R` 로 변환(매핑)             | `map(Function)`                       |
| `Consumer<T>`    | `void accept(T t)`  | `T` 를 받아 소비(반환값 없음) (Side-effect) | `forEach(Consumer)`, `peek(Consumer)` |
| `Supplier<T>`    | `T get()`           | 인자 없이 `T` 생성/제공 (공급)              | `collect`, `generate`                 |
추상 메서드란 인터페이스들의 구현되지 않은 메서드임
함수형 인터페이스는 실행될 때 입력을 받아서 이런 출력을 내놓아야 한다는 규격을 정의함

매핑 메서드란 인터페이스에 정의된 메서드로서 함수형 인터페이스를 매개변수로 받아서, 스트림의 처리 과정에 등록 하는 역할임. 학술적으로 고차 함수 라고 부른다.

# 스트림 파이프라인 아키텍처

## 파이프라인 3단계 구조
스트림이라면 반드시 거치는 3단계를 뜻함.
`Source` -> `Intermediate Operations` -> `Terminal Operation`
소스, 중간 연산, 종단 연산

흔히
데이터의 원천인 소스를 가지고 출발해서,
중간 연산인 `filter`, `map`, `sorted` 등을 거쳐
종단 연산인 `collect`, `forEach`, `reduce` 등을 수행한다고 생각하지만
그렇지 않음.
종단 연산이 호출되기 전까지 전혀 아무런 연산을 안함.

이 과정의 핵심 기술은 `Lazy Evaluation` 지연 연산임

`list.stream().map(...).filter(...)`

`stream()` 을 호출하면 스트림의 Head 가 생성됨.
`map()` 을 호출하면 이전 단계를 가리키는 새로운 `Stage` 객체가 생성되어 연결됨.
`filter()` 를 호출하면 앞서 `map` 을 가리키는 또 다른 `Stage` 객체가 생성되어 연결됨.

데이터 소스를 가공하기 시작하는게 아니라 head 를 시작으로 무슨 작업을 해야하는지 기록만 하고 리턴함. 종단 연산이 호출될 때까지. 그래서 Lazy 인거임

```java
List<String> names = Arrays.asList("Java", "Kotlin", "Scala");

Stream<String> stream = names.stream()
	.filter(name -> {
		sout("Filtering: " + name);
		return name.length() > 3;
	})
	.map(name -> {
		sout("Mapping: " + name);
		return name.toUpperCase();
	});
	
sout("파이프라인 지연 로딩 테스트");
```

만약 stream 이 그냥 순서대로 정해진 작업을 진행해 나가는 코드였다면,
위 예제를 실행시켰을때
단계마다 sout이 실행될 것임.
하지만 실제로는 "파이프라인 지연 로딩 테스트" 만 보이게 됨. (종단 연산이 선언이 안되어 있음)

```java
List<String> names = Arrays.asList("Java", "Kotlin", "Scala");

Stream<String> stream = names.stream()
	.filter(name -> {
		sout("Filtering: " + name);
		return name.length() > 3;
	})
	.map(name -> {
		sout("Mapping: " + name);
		return name.toUpperCase();
	});
	
sout("파이프라인 지연 로딩 테스트");

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

최종적으로 종단 연산을 호출하면 그 때부터 순차적으로 기록해둔 작업이 실행됨.
`Eager Evaluation` 조급한 연산(?) 이라고도 표현함

종단 연산이 호출되면 라이브러리는 그제서야 앞서 구축한 파이프라인을 역추적하며 수행 계획을 파악하고 데이터 소스에게 데이터를 보내라고 요청함.
데이터가 흐르며 정의된 중간 연산들을 실행하며 통과함

## Stream 제공 연산 총 정리

- 중간 연산은 상태의 유무에 따라 두 가지로 나뉜다.

1. 무상태 연산, Stateless Operations
이전 연산의 데이터를 기억하지 않고 그냥 바로 처리해서 넘겨줄 수 있는 연산
메모리 부하가 거의 없어 데이터 양이 아무리 많아도 처리 가능하다

`filter(Predicate)`, 조건에 맞는 것만 통과 시킴
`map(Function)`, 데이터를 변환 시킴
`flatMap(Function)`, 스트림 평탄화 2차원배열을 1차원으로
`peek(Consumer)`, 데이터 흐름 확인, 디버깅용
`unordered()`, 순서가 중요하지 않음을 명시하여 병렬 처리를 최적화함

2. 상태 유무 연산, Stateful Operation
데이터 처리를 위해 현재 상태를 저장할 필요가 있는 연산들
버퍼를 사용해서 상태를 저장하기 때문에 메모리를 많이 사용하고 전체 데이터를 다 순회할때까지 다음 단계로 데이터를 넘기지 못할 수 있음

`sorted(Comparator)`, 데이터를 정렬하기 위해 모든 데이터를 다 모아야하기 때문에 가장 비싼 연산이다
`distinct()`, 중복을 제거하는 연산, 말 그대로 중복 여부를 알기 위해 상태를 저장해야할 필요가 있다
`limit(long)`, 개수 제한, 카운트 상태를 유지해야 한다
`skip(long)`, 건너뛰기


- 종단 연산, 반환 타입이 Stream 이 아니게 된다. 파이프라인이 닫히고 결과가 나오는 단계.
단락(short-circuiting) 여부와 결과의 형태에 따라 분류된다.

1. 리덕션, 데이터를 축소해서 하나의 결과를 만들어냄
`collect(Collector)`, List, Map, Set 등으로 결과를 도출
`reduce(BinaryOperator)`, 누적 계산, 합계 구하기 같은 것
`count()`, 결과의 개수를 반환
`max(Comparator)`, `min(Comparator)`, 최대값, 최소값 구하기

2. 순회 및 부작용(?), 리턴값이 없거나 로직 수행 자체가 목적인 경우
`forEach(Consumer)`, 각 요소에 대해 반복 수행, `forEach` 안에 로직을 짜지 말것. 출력이나 외부 저장 용도로만 사용
`forEachOrdered(Consumer)`, 병렬 수행에서도 순서를 보장하며 순회

3. 매칭 및 검색 - 쇼트 서킷, 모든 데이터를 보지 않아도 결과를 낼 수 있는 연산
당연히 성능이 올라간다.
`anyMatch(Predicate)`, 하나라도 만족하면 즉시 `true` 반환 후 종료
`allMatch(Predicate)`, 모두 만족해야 `true`, 하나라도 틀리면 즉시 `false` 후 종료
`noneMatch(Predicate)`, 모두 만족하지 않으면 `true`
`findFirst()`, 첫번째 요소 반환
`findAny()`, 아무거나 하나 반환, 병렬처리에서 빨라짐



# 루프 퓨전(loop funsion)과 싱크(sink)

## 멀티 패스와 싱글 패스

```java
List<Integer> temp1 = new ArrayList<>();
for (Integer i : list) {
	if (i > 3) {
		temp1.add(i);
	}
}

List<Integer> temp2 = new ArrayList<>();
for (Integer i : temp1) {
	temp2.add(i * 2);
}

for (Integer i : temp2) {
	System.out.println(i);
}
```

이것이 멀티 패스다.
스트림이 멀티 패스로 작동한다고 하면 매 호출마다 컬렉션을 계속 순회해야한다. 이건 엄청난 비효율이 발생한다.

실제 스트림은 싱글 패스로 작동한다.

```java
for (Integer i : list) {
	if (i > 3) {
		int mapped = i * 2;
		System.out.println(mapped);
	}
}
// 실제 동작은 이렇게 '루프 퓨전' 해서 작동한다
```

이제 어떻게 서로 다른 객체인 필터나 맵 등이 하나로 뭉쳐지는지 알아보자

## 핵심은 싱크
`java.util.stream.Sink` 라는 인터페이스가 핵심이다.

```java
interface Sink<T> extends Consumer<T> {
    // 1. 순회 시작 전 호출 (초기화)
    default void begin(long size) {}

    // 2. 순회 끝난 후 호출 (정리)
    default void end() {}

    // 3. 쇼트 서킷을 위한 중단 신호 (예: findFirst, limit)
    default boolean cancellationRequested() { return false; }

    // 4. 실제 데이터 처리 (Consumer의 메서드)
    void accept(T t);
}
```

`Consumer` 를 확장한 인터페이스로 데이터 처리 상태를 관리하는 메서드가 선언되어 있다.

이제 Stream 은 종단 연산이 호출되면 파이프라인 맨 뒤에서부터 앞으로 거슬러 올라가며 `Sink` 들을 연결한다.
`Terminal Sink` -> `Map Sink` -> `Filter Sink`
즉, `FilterSink(MapSink(TerminalSink))` 구조가 된다.

`stream.filter(x -> x > 3).map(x -> x * 2).forEach(System.out::println);`

```java
// 1. 가장 안쪽의 Sink (Downstream) - forEach 단계
Sink<Integer> terminalSink = new Sink<>() {
    public void accept(Integer i) {
        System.out.println(i); // 최종 출력
    }
};

// 2. 그 위를 감싸는 Sink - Map 단계
Sink<Integer> mapSink = new Sink<>() {
    public void accept(Integer i) {
        int result = i * 2;    // [Map 로직 수행]
        terminalSink.accept(result); // 다음 단계(Downstream)로 토스
    }
};

// 3. 가장 바깥쪽 Sink - Filter 단계
Sink<Integer> filterSink = new Sink<>() {
    public void accept(Integer i) {
        if (i > 3) {           // [Filter 로직 수행]
            mapSink.accept(i); // 통과하면 다음 단계(Downstream)로 토스
        }
        // 통과 못하면? 아무것도 호출 안 함(Drop).
    }
};
```

개발자가 넘겨준 람다가 역순으로 Sink가 생성되며 각각 포함된다.
forEach -> map -> filter
중간 연산은 각각 자기 다음에 실행될 연산을 호출하도록 생성된다.
실행 단계에서 데이터 소스를 넘겨줄때 다시 정방향으로 호출되며 이 때 단 하나의 스레드, 하나의 호출 스택 안에서 연쇄적으로 일어나게 된다. 싱글패스.
FilterSink.accept(i) 를 통과하면
MapSink.accept(i) 반환되면
TerminalSink.accept(i) 출력.

## 쇼트 서킷
`Sink` 인터페이스의 `cancellationRequested()` 메서드가 이 기능을 구현하는 메서드다.

```java
Sink<Integer> limitSink = new Sink<>() {
	int count = 0;
	
	public void accept(Integer i) {
		if (count < 3) {
			count++;
			downstream.accept(i);
		}
	}
	
	public boolean cancellationRequested() {
		return count >= 3;
	}
}
```

데이터 소스에서 데이터를 하나씩 넘길 때마다 Sink.cancellationRequested() 메서드를 실행시킨다.
조건에 맞아 참이 반환되면 반복이 종료되고 불필요한 연산을 방지한다.

```java
while (!sink.cancellationRequested()) { // 상시 체크
	boolean hasMoreData = spliterator.tryAdvance(sink);
	
	if (!hasMoreData) {
		break;
	}
}
```





## Spliterator

`Iterator` 인터페이스는 Java 8 까지 컬렉션 순회의 표준이었는데 태생적으로 순차적으로 구현되어 있어 데이터를 나눌 수가 없음. 이를 개선한게 Spliterator 인터페이스임.

분할 + 반복자 = Spliterator

데이터 소스를 순회하고 필요할 경우 분할 하여 병렬 처리하는게 목적임.
모든 Collection 이 Stream 호출 시 곧바로 내부적으로 자기 구조에 맞는 Spliterator 구현체를 생성해서 리턴함

### 핵심 메서드
- `tryAdvance(Consumer action)`
순차 처리를 담당하는 메서드.
요소 하나를 가져와서 `Consumer` 인 `Sink` 에게 전달하고 다음 요소가 있으면 참 없으면 거짓을 반환

`while(spliterator.tryAdvance(sink)) {...}`

- `trySplit()`
병렬 처리를 담당하는 메서드
1. 자신이 가진 데이터의 일부(보통 절반으로 함)를 떼어내서 새로운 Spliterator 객체를 만들어 반환
2. 자신은 나머지 절반만 관리
3. 더 쪼갤 수 없을 정도로 데이터가 적다면 null 반환


### 데이터 분할 성능

병렬 스트림이 호출되면 자바의 ForkJoinFramework 는 `trySplit()` 을 재귀적으로 호출함

- 인덱싱 기반, Random Access 자료구조의 경우
	- ArrayList, Arrays.stream, Vector: 최상의 효율
		- 메모리에 연속할당된 자료구조로 mid = (start + end) / 2 로 중간점을 바로 구함
		- O(1), 데이터 크기와 상관없이 인덱스로 순식간에 절반 쪼개짐
		- 병렬 처리 시 스레드 간의 작업 분할이 완벽하게 균등하게 됨
	- HashSet, HashMap: 좋음
		- 내부에 버킷이라는 배열을 가짐, 버킷 안에 노드들이 있는 구현 형태
		- 버킷 배열을 절반으로 나누게되는데
		- O(1)이긴 한데 버킷이 어떤건 많고 어떤건 비어있을 가능성도 있어서 ArrayList 만큼 균등하진 않다.
		- 버킷마다 데이터가 많을수도, 적을수도 있기 때문에 ArrayList 만큼 완벽하지 않지만 꽤 효율적인 병렬처리가 이뤄짐
	- 

- 노드 기반, 순차 접근, Sequential Access 자료구조의 경우
	- LinkedList: 최악 효율
		- 메모리에 흩어진 채로 노드들이 주소로 연결됨
		- 분할 지점을 찾기 위해 N/2 번 이동이 필요해짐
		- O(N) 쪼개는거부터 엄청난 비용이 든다
		- 쪼개는데 시간을 다 쓰고, 균등하게 나눠지지도 않는다.
	- TreeSet, TreeMap: 좋음
		- 노드가 왼쪽 오른쪽으로 구분됨
		- 루트 노드를 기준으로 왼쪽 트리와 오른쪽 트리로 쪼갤 수 있음
		- 분할 방식이 비교적 저렴해지고, 트리가 완전 균등하진 않지만 어느정도 균등을 보장해줌

| **효율** | **자료구조**                                | **분할 비용** | **데이터 균등성** | **추천 여부**  |
| ------ | --------------------------------------- | --------- | ----------- | ---------- |
| **최고** | `ArrayList`, `Array`, `IntStream.range` | **O(1)**  | **Perfect** | **강력 추천**  |
| **좋음** | `HashSet`, `HashMap`                    | O(1)      | Good        | 추천         |
| **좋음** | `TreeSet`, `TreeMap`                    | Low       | Very Good   | 추천         |
| **나쁨** | `LinkedList`, `IO Stream`               | **O(N)**  | **Bad**     | **절대 비추천** |



### 특정 상수 
Spliterator 는 단순 데이터만 주는게 아니라 데이터 소스의 특성을 스트림 엔진에게 알려줌.
이 정보를 토대로 내부 최적화가 수행됨.

`characteristics()` 메서드로 비트 마스크 형태의 상수를 반환함

|**특성 상수**|**의미**|**최적화 영향 (Optimization)**|
|---|---|---|
|**`SIZED`**|크기가 정확함 (Array, List)|배열을 미리 할당하거나 분할 계획을 정확히 세움.|
|**`SUBSIZED`**|분할된 조각도 크기가 정확함|병렬 처리 시 작업 부하 균형(Load Balancing)이 완벽함.|
|**`ORDERED`**|순서가 있음 (List, SortedSet)|`limit`, `skip` 등의 연산이 순서대로 동작함을 보장.|
|**`SORTED`**|이미 정렬되어 있음|`stream.sorted()` 호출 시 정렬 작업을 생략(Skip)함. (O(N log N) -> O(1))|
|**`DISTINCT`**|중복이 없음 (Set)|`stream.distinct()` 호출 시 검사 작업을 생략함.|



# 병렬 스트림 성능

## 실행 엔진
`ForkJoinPool`, `Work-Stealing`

병렬 스트림은 단순히 스레드 여러개를 만들어 돌리는게 아님
Java 7  에서 도입된 `Fork/Join Framework` 로 동작함

### Common Pool
공용 풀.
별도 설정이 없으면 병렬 스트림은 JVM 에 단 하나만 있는 `ForkJoinPool.common` 을 공유하게 됨.
스레드 개수는 CPU 코어 수 - 1 로 기본 설정 되어 있음
즉, 하나의 병렬 스트림이 네트워크 I/O 등으로 인해 블로킹 되면 애플리케이션 전체의 병렬 스트림이 멈추는 대형 장애로 이어질 수 있음

### Work-Stealing 알고리즘
병렬 스트림 효율의 핵심

각 스레드가 `Deque` 작업 큐를 가짐.
Spliterator 가 쪼갠 작업들이 각 스레드의 큐에 쌓임
어떤 스레드가 자기 일이 먼저 끝나면, 다른 바쁜 스레드 큐의 뒷 부분에서 작업을 훔쳐와 처리하기 시작함.
스레드가 놀지 않고 효율을 최대로 올리는 알고리즘.


## 병렬화 오버헤드
그럼 무조건 병렬 처리를 하면 빨라지는가? no
병렬화에도 비용이 있음.

총 처리 시간 = 분할하는데 드는 비용 + 실제 작업 처리 시간 + 나눠진 결과물 병합 비용 + 스레드 간 컨텍스트 스위칭 및 캐시 미스 비용

'실제 작업 처리 시간' 은 본래 '싱글 스레드 실제 작업 처리 시간' 의 N분이 될 것임.
즉, 나머지 코스트를 다 합쳤을 때 '싱글 스레드 실제 작업 처리 시간' 보다 짧아야 병렬화의 의미가 생김.

>그걸 어떻게 아는데?

### NQ 모델
Java 설계자 Brian Goetz 가 제시한 모델로 병렬 스트림 사용 여부 결정

`N * Q > 10,000`, 10,000 이 넘으면 병렬화 해라.
N : 데이터의 개수
Q : 요소 하나를 처리하는데 드는 비용

N이 작을 때, 데이터가 1,000 개 이하라면 병렬화의 오버헤드가 더 커져버림. 그냥 순차 스트림 써야함

Q가 매우 클 때, 즉 N이 작아도 각각의 처리 비용이 매우 비싸면 병렬 처리하는게 좋음. 복잡한 암호화 연산 등이 여기에 해당함

`int` 덧셈 같은 연산은 아주 간단한 연산일 거임. Q가 엄청나게 작다면 N이 어마무시하게 많아야 병렬화의 효과를 볼 수 있다는 소리임.

### 병합 비용
합치는 비용도 중요함. 쪼갠만큼 합쳐야하니까.

sum(), max(), min(), count() 등은 숫자끼리 더하거나 비교하니까 가격이 매우 쌈. 병렬화에 유리.

collect(Collectors.toSet()), Collectors.toMap() 등은 병합 비용이 비쌈.
서로 다른 두 개의 set 이나 map 을 하나로 합치는 과정은 내부적으로 엄청난 데이터 복사와 재해싱이 발생함. 이러면 병렬 처리가 순차 처리보다 느려질 수도 있음.

### 판단 기준 요약
1. 자료구조가 분할에 효율적인가?
2. 요소의 수가 충분히 많은가? NQ에 대입하거나, 최소 1만 개 이상인지 확인해보자.
3. 각 요소는 독립적인가? 상태를 공유하지 않으면 Race Condition이 없다
4. 병합 비용이 저렴한가?
5. 박싱,언박싱 비용이 적은가? 기본형 스트림을 사용하는지 여부


