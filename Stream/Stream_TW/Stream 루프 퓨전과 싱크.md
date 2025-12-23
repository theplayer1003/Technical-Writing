# Stream 루프 퓨전과 싱크

# 멀티 패스와 싱글 패스
일반적인 컬렉션의 처리는 멀티 패스 구조를 가진다.

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

처리하고자 하는 작업 단계마다 중간 컬렉션을 생성하고 순회하기 때문에 데이터 N개에 대해 연산 단계가 M개라면 N * M 번의 순회가 필요해 비효율적이다.

Stream 은 싱글 패스 구조로 '루프 퓨전' 해서 작동한다.

```java
for (Integer i : list) {
	if (i > 3) {
		int mapped = i * 2;
		System.out.println(mapped);
	}
}
```

하나의 `for` 문 안에 로직을 합친 것처럼 동작하여 N개의 데이터 개수만큼 순회한다. 어떻게 서로 다른 객체(`filter`, `map`), 연산들이 하나의 파이프라인으로 통합될까?

# 싱크
루프 퓨전의 구현은 `java.util.stream.Sink` 라는 인터페이스가 핵심이다.

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

`Sink<T>` 는 데이터를 받아 처리하고 다음 단계로 넘겨주는 역할을 한다. `Consumer<T>` 를 확장해 상태를 관리하는 메서드가 선언되어 있다.

이를 통해 멀티 패스처럼 중간 컬렉션을 만드는 대신, 처리된 데이터를 즉시 다음 단계의 `Sink` 로 전달(`accept`)이 가능하다.

종단 연산이 호출되면 파이프라인 맨 뒤에서부터 앞으로 거슬러 올라가 `Sink` 를 연결한다.

`Terminal Sink` -> `Map Sink` -> `Filter Sink`
`FilterSink(MapSink(TerminalSink))`
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

- 실제 실행 흐름
1. `Spliterator`(공급자) 가 데이터 소스에서 요소 하나를 꺼낸다
2. 제일 바깥쪽인 `filterSink` 의 `accept()` 를 호출한다
3. 조건이 통과되면 `mapSink.accept()` 를 호출, 통과하지 못하면 버려진다
4. 마찬가지로 로직을 수행하고 다음 단계로 전달해 최종 출력된다

이 모든 과정이 단 하나의 스레드 스택에서 일어난다. Loop Fusion.

## 쇼트 서킷
쇼트 서킷은 무한 스트림이나 대용량 데이터 처리 시, 모든 데이터를 순회하지 않고 멈추는 기능이다.
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

`Spliterator` 는 항상 데이터를 `Sink` 에 넘기기(`tryAdvance`) 전에 중단 요청이 있었는지 먼저 확인한다.

```java
while (!sink.cancellationRequested()) { // 상시 체크
	boolean hasMoreData = spliterator.tryAdvance(sink);
	
	if (!hasMoreData) {
		break;
	}
}
```

조건을 검사하고 필요한 경우 반복을 종료해 불필요한 연산을 방지한다.