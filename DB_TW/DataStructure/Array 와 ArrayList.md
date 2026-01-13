# 내부 구조 및 성능 최적화

# 개요
자바의 대표적인 선형 자료구조 Array, ArrayList 의 내부 동작 원리를 학습한다.


# Array
Array 는 논리적 순서와 물리적(메모리) 순서가 일치하는 자료구조이다. 단순히 자료를 담은 통이 아니라 동일한 타입의 데이터가 연속된 메모리 공간에 저장된다.

## Random Access
Array 는 인덱스를 통해 O(1) 의 시간 복잡도로 데이터에 접근한다. 자료구조를 순회하며 탐색하는 방식이 아닌 인덱스를 통해 주소를 계산하는 방식으로 접근할 수 있기 때문이다.

>Memory Address = Base Address + (Index * Data Size)

- 단점
Array 는 처음 생성 시 힙 영역에 고정된 크기의 메모리 블록을 할당 받는다. 따라서 런타임에 이 크기를 변경할 수 없다.
초기 할당 크기가 실제 데이터보다 크면 메모리 낭비가 발생하며, 적으면 데이터를 담지 못하기 때문에 성능 최적화를 위해 크기 할당을 신경 써야한다.

>[!tip] 여기서 Random 은 '무작위' 가 아닌 '임의' 라는 의미를 가진다
>'어떤 임의의(Random) 위치'를 찍더라도 접근 시간이 일정하다는 의미에서 Random Access


# ArrayList
ArrayList 는 Array 의 고정된 크기에서 오는 한계를 극복하기 위해 나온 Wrapper 클래스 다.
내부적으로 `Object[]` 를 통해 자료를 저장하며 데이터 양에 따라 스스로 크기를 조정하는 동적 리사이징(Dynamic Resizing) 이 구현되어 있다.

```java
public class ArrayList<E> extends AbstractList<E> ... {
	transient Object[] elementData;
	
	private int size;
	
	private static final int DEFAULT_CAPACITY = 10;
}
```

- `elementData` 는 실제 데이터를 담을 배열
- `size` 는 현재 저장된 데이터의 개수
- `Capacity` 는 `elementData` 배열의 물리적 크기, 따로 지정해주지 않으면 기본 값을 사용

## Dynamic Resizing
데이터 추가 요청이 들어왔는데 현재 저장된 데이터의 개수와 용량이 같다면 (`size == capacity`) 용량을 확장하게 된다. 보통 1.5배의 새로운 배열을 힙 메모리에 할당한다.
`newCapacity = oldCapacity + (oldCapacity >> 1)`

이후 기존 데이터를 새로운 배열에 옮기기 위해 O(N) 의 비용이 발생한다.

## System.arraycopy 성능 최적화
앞서 언급한 리사이징이나 데이터들의 중간에서 이루어지는 삽입, 삭제가 발생하면 데이터의 이동이 필요하다.
ArrayList 는 이 작업을 위해 단순 반복문을 돌며 처리하지 않고 `System.arraycopy` 메서드를 사용해 성능을 최적화한다.

```java
public static native void arraycopy(Object src, int srcPos, Object dest, int destPos, int length);
```

이는 Java Native Interface(JNI) 의 메서드로 C/C++ 함수인 `memcpy`, `memmove` 를 직접 호출한다. 메모리 블록 자체를 통째로 이동시키는 방식으로 CPU의 벡터 명령어(SIMD) 등을 활용하기 때문에 대량의 데이터를 반복문으로 처리하는 것 보다 압도적인 성능을 보여준다.

## 시간 복잡도 및 성능 분석
| 연산      | 시간 복잡도         | 설명                                      |
| ------- | -------------- | --------------------------------------- |
| 조회(get) | O(1)           | 인덱스 연산을 통한 즉시 접근                        |
| 추가(add) | Amortized O(1) | 리사이징이 발생하지 않으면 O(1), 발생하는 경우에만 O(N)     |
| 삽입/삭제   | O(N)           | 중간에서 발생 시 해당 요소 이후의 모든 요소를 이동시키는 작업이 필요 |

## 실무 적용 가이드

- 초기 용량 설정
데이터의 크기를 예상할 수 있다면 초기 용량을 직접 지정해줘야 한다.
기본 값으로 생성 후 100만 개의 데이터를 삽입 한다고 가정하면 매우 큰 성능 손해가 발생한다. 미리 초기 용량을 지정해 리사이징 횟수를 최소화 시키자.

- `Arrays.asList()`
이 메서드는 `java.util.ArrayList` 를 반환하지 않는다. `Arrays` 클래스 내부의 정적 클래스 `private static class ArrayList` 를 반환하며 내부 배열을 원본 배열과 공유한다.
이 리스트는 add, remove 등의 호출이 불가능하며 (`UnsupportedOperationException`), 수정이 필요할 경우 `new ArrayList<>(Arrays.asList(...))` 로 감싸서 새 인스턴스를 생성해야 한다.