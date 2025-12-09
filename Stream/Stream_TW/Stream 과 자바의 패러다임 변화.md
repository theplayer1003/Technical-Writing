# Stream 과 자바의 패러다임 변화

# 개요
- 명령형에서 선언형으로
Stream API 는 Java 8 에서 데이터 처리 연산을 파이프라인으로 연결해 유연하게 처리하는 기술이다.
이 도입을 통해 프로그래밍 패러다임을 명령형에서 선언형으로 전환해 코드의 가독성과 재사용성을 향상시켰다.

## 명령형-How
명령형이란 개발자가 컴퓨터가 무엇을 어떻게 수행해야하는지 명시하는 방식이다.

```java
for (int i = 0 ; i < list.size() ; i++) {
	// 작업 A
	// 작업 B
	// 작업 C
}
```

이것이 명령형이다. 어떤 작업을 어떤 순서대로 반복해야할지 개발자가 명시해준다.
개발자가 어떻게 처리할지 세부적으로 과정을 제어한다.

## 선언형-What
선언형이란 개발자가 컴퓨터가 무엇을 수행해야하는지만 명시하는 방식이다.

```SQL
SELECT * FROM users WHERE id = 1;
```

제일 대표적인 선언형 방식의 예로 SQL 문을 들 수 있다.
SQL 문은 개발자가 'users 정보 중에 id가 1인 데이터만 찾아라'(무엇) 라는 목표만 정해주고 '어떻게' 수행할지에 대한 정보가 빠져있다.
'어떻게' 는 DBMS 가 내부적으로 알아서 처리한다.

이를 통해 개발자는 비즈니스 로직의 목적에만 집중할 수 있다.

# 외부 반복, 내부 반복
- 두 패러다임을 외부 반복과 내부 반복으로 알아보자

## 외부 반복

```java
List<String> names = Arrays.asList("Pobi", "Crong", "James");
for (String name : names) {
	System.out.println(name);
}

// 컴파일 후 이렇게 바뀜
Iterator<String> iterator = names.iterator();
while (iterator.hasNext()) {
	String name = iterator.next();
	System.out.println(name);
}
```

외부 반복은 명령형 패러다임에 속하며 개발자가

어떤 작업(`System.out.println(name)`)을
어떻게 수행할지(`Iterator 를 이용해 while() 문으로 순회하며 반복`)

명시하고 있다.

- **단점 1 (강한 결합)**
데이터를 가져오는 행위와 데이터를 처리하는 행위 모두가 코드에 섞여 있다.

- **단점 2 (병렬화의 어려움)**
순차적인 처리 방식이 코드에 고정되어 있다.

### 병렬화가 어려운 이유
- 외부 반복(명령형)이 어떻게 병렬화가 어려운지 알아보자

```java
long sum = 0;
for (int i = 0; i< list.size();i++) {
	sum += list.get(i);
}
```

 위 코드를 병렬화 하려면 다음과 같은 작업이 필요하다.

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

1. 작업을 분할한다
2. 스레드에 할당한다
3. 각 스레드를 시작시킨다
4. 각 결과를 기다렸다 병합한다

본래의 비즈니스 로직보다 병렬화 처리를 위한 코드가 더 많아진걸 알 수 있다.

## 내부 반복

```java
List<String> names = Arrays.asList("Pobi", "Crong", "James");

names.stream().forEach(name -> System.out.println(name));
```

내부 반복은 선언형 패러다임에 속하며 개발자는 어떤 작업(`System.out.println(name))`을 수행해야하는지만 명시한다.

명령형에 있던 코드의 제어권 `next()`, `hasNext()` 등의 코드가 라이브러리 안으로 이동해 JVM이 하드웨어 아키텍처를 활용해 병렬 실행 여부나 최적화 전략을 스스로 결정하도록 한다.

```java
long sum = list.stream()
			.parallel() // '병렬화 해'. 끝.
			.mapToLong(i -> i)
			.sum();
```

병렬화 작업 역시 선언형으로 간단하게 처리 할 수 있다.

# 함수형 인터페이스와 람다
내부 반복을 위해서는 Stream API 에게 수행할 동작을 전달해야 한다.
자바는 객체지향 언어로 함수 자체를 메서드의 인자로 넘길 수 없는 제약이 있었다.

이 문제를 해결하기 위해 나온게 `함수형 인터페이스` 이다. 이는 수행할 동작을 담기 위한 그릇이며 람다 표현식을 통해 코드를 간단하게 전달할 수 있게 해준다.

```java
@FunctionalInterface
public interface MyFunction {
	void run();
}
```

# Stream API
- 4대 핵심 인터페이스

| 인터페이스            | 추상메서드               | 역할                                | Stream 매핑 메서드                         |
| ---------------- | ------------------- | --------------------------------- | ------------------------------------- |
| `Predicate<T>`   | `boolean test(T t)` | `T` 를 받아 `boolean` 반환(판별)         | `filter(Predicate)`                   |
| `Function<T, R>` | `R apply(T t)`      | `T` 를 받아 `R` 로 변환(매핑)             | `map(Function)`                       |
| `Consumer<T>`    | `void accept(T t)`  | `T` 를 받아 소비(반환값 없음) (Side-effect) | `forEach(Consumer)`, `peek(Consumer)` |
| `Supplier<T>`    | `T get()`           | 인자 없이 `T` 생성/제공 (공급)              | `collect`, `generate`                 |
