# ConcurrentHashMap의 내부 구현과 성능 분석

`ConcurrentHashMap`은 자바에서 동시성 문제를 해결하기 위해 제공되는 스레드 안전한 Map 구현체로, `HashMap`을 동시성 환경에서 안전하게 사용할 수 있도록 여러 최적화 기법을 적용하여 설계되었다. `ConcurrentHashMap`의 내부 구조, 분할 잠금(Segment Lock)과 CAS(Compare And Swap) 메커니즘을 살펴보고, `HashMap`, `synchronizedMap`과의 성능을 비교해본다.

## 1. ConcurrentHashMap의 내부 구조

`ConcurrentHashMap`은 자바 1.5에서 처음 도입되었으며, 자바 8부터는 그 내부 구조에 많은 개선이 이루어졌다. 초기 버전에서는 분할 잠금 방식을 사용해 여러 쓰레드가 동시에 `put` 및 `get` 작업을 수행할 수 있도록 하였다. 자바 8부터는 분할 잠금 방식 대신 **배열과 연결 리스트 또는 트리(레드-블랙 트리)**를 혼합한 구조로 구현되어, 성능이 크게 향상되었다.

- **배열과 연결 리스트/트리 구조**: 기본적으로 `ConcurrentHashMap`은 배열 기반으로 데이터를 저장하며, 해시 충돌이 발생하면 연결 리스트로 데이터를 관리한다. 데이터가 일정 개수 이상 쌓이면 연결 리스트는 **레드-블랙 트리**로 전환되어 검색 성능을 O(log n)으로 향상시킨다.
- **분할 잠금 (Bucket Locking)**: 자바 8 이전에는 `Segment`라는 내부 클래스를 통해 여러 개의 버킷을 나누고, 각 버킷에 대해 별도의 잠금을 걸어 다수의 쓰레드가 동시에 접근할 수 있도록 하였다. 자바 8 이후에는 **각 해시 버킷에 대해 개별적으로 동기화**하여 동시성을 보장하는 방식으로 변경되었다.

## 2. CAS (Compare And Swap) 활용

`ConcurrentHashMap`은 **CAS(Compare And Swap)** 메커니즘을 사용하여 특정 상황에서 락을 사용하지 않고도 안전하게 값을 갱신할 수 있도록 한다. CAS는 기존 값을 읽어와 예상 값과 비교한 후, 동일할 경우에만 새로운 값으로 변경하는 원자적 연산이다. 이를 통해 락을 사용하는 경우 발생할 수 있는 **병목 현상**을 줄이고, 성능을 극대화한다.

예를 들어, `putIfAbsent()` 메소드는 기존에 해당 키가 존재하지 않을 때만 값을 추가하는데, 이 과정에서 CAS를 사용하여 안전하게 값을 삽입한다. 이러한 방식은 특히 **락의 경쟁이 심한 환경**에서 성능 이점을 제공한다.

## 3. ConcurrentHashMap vs. HashMap vs. synchronizedMap 성능 비교

### HashMap

`HashMap`은 스레드 안전하지 않은 일반적인 Map 구현체로, 단일 쓰레드 환경에서는 가장 빠른 성능을 제공한다. 하지만 **멀티 쓰레드 환경**에서 여러 쓰레드가 동시에 `HashMap`을 수정할 경우 **데이터 불일치** 문제가 발생할 수 있다. 따라서 동시성 제어가 필요한 경우 적절한 락을 사용해야 한다.

### synchronizedMap

`synchronizedMap`은 `Collections.synchronizedMap()` 메서드를 통해 `Map` 인터페이스를 스레드 안전하게 만드는 래퍼 클래스이다. 내부적으로 모든 접근에 대해 **synchronized 블록**을 사용하여 락을 걸기 때문에, **경합이 심한 환경**에서는 성능이 크게 저하될 수 있다. 모든 쓰레드가 동일한 락을 기다려야 하므로, **락의 경쟁**이 많은 경우 병목이 발생할 수 있다.

### ConcurrentHashMap

`ConcurrentHashMap`은 위 두 가지 Map과 비교했을 때, **멀티 쓰레드 환경에서 최고의 성능**을 제공한다. `synchronizedMap`과 달리 전체 맵에 락을 거는 것이 아니라, **필요한 부분에만 락을 거는 방식**을 사용하기 때문에 락의 경쟁을 최소화하고, 여러 쓰레드가 동시에 Map을 수정할 수 있다. 또한, CAS를 사용하여 일부 연산을 락 없이 처리함으로써 **락 경합**을 더욱 줄일 수 있다.

실제로 많은 쓰레드가 동시에 읽기와 쓰기를 수행하는 환경에서 `ConcurrentHashMap`은 **거의 선형적인 확장성**을 보여주며, 읽기 작업과 쓰기 작업이 빈번하게 섞여 있을 때도 **안정적인 성능**을 유지한다.

## 4. ConcurrentHashMap의 활용 예시

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.Map;

public class ConcurrentHashMapExample {
    public static void main(String[] args) {
        Map<String, Integer> map = new ConcurrentHashMap<>();
        
        // 값 추가
        map.put("apple", 1);
        map.put("banana", 2);
        
        // 값 조회 및 갱신
        map.computeIfPresent("apple", (key, value) -> value + 1);
        
        // 값 삽입, 키가 없을 경우에만
        map.putIfAbsent("cherry", 3);
        
        System.out.println(map);
    }
}
```

위의 예제에서는 `ConcurrentHashMap`을 사용하여 다수의 쓰레드가 동시에 접근해도 안전한 Map을 구현하였다. `putIfAbsent()`와 `computeIfPresent()` 등의 메서드는 락의 사용을 최소화하면서도 스레드 안전한 연산을 제공한다.

## 마무리

`ConcurrentHashMap`은 멀티 쓰레드 환경에서 안전하게 사용할 수 있는 Map 구현체로, 분할 잠금과 CAS 메커니즘을 통해 **최대의 성능**을 제공한다. `HashMap`이나 `synchronizedMap`과 비교했을 때 동시성 환경에서 성능과 안정성 면에서 큰 장점을 지닌다. 따라서, **다수의 쓰레드가 동시에 접근하는 상황**에서는 `ConcurrentHashMap`을 사용하는 것이 권장된다. 다만, 불필요한 락 사용을 피하고 최대한 읽기와 쓰기를 병행하여 처리할 수 있는 구조를 설계하는 것이 중요하다.

