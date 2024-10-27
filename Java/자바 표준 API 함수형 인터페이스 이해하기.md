# Consumer, Supplier, Function, Operator, Predicate

자바 8부터 함수형 프로그래밍 스타일이 대거 도입되면서, 람다 표현식과 함께 사용할 수 있는 여러 가지 함수형 인터페이스가 등장했다. 자바 8의 대표적인 함수형 인터페이스인 **Consumer**, **Supplier**, **Function**, **Operator**, **Predicate** 에 대해 알아보고, 각 인터페이스를 어떻게 활용할 수 있는지 코드 예제와 함께 확인해본다.

## 1. Consumer

**Consumer**는 입력 값을 받아서 어떤 동작을 수행하지만, 결과를 반환하지 않는 함수형 인터페이스로 주로 데이터를 소비(Consumer)하는 역할을 한다.

**메소드 시그니처**: 
```java
void accept(T t)
```

**예제**:
```java
import java.util.function.Consumer;

public class ConsumerExample {
    public static void main(String[] args) {
        Consumer<String> greeter = name -> System.out.println("Hello, " + name + "!");
        greeter.accept("World");
    }
}
```

위 코드에서는 `Consumer` 인터페이스를 사용하여 입력 값 `name`을 받아 콘솔에 인사말을 출력하며, 반환 값은 없다.

## 2. Supplier

**Supplier**는 인자를 받지 않고 값을 반환하는 함수형 인터페이스로 주로 데이터를 제공(Supply)하는 역할을 한다.

**메소드 시그니처**: 
```java
T get()
```

**예제**:
```java
import java.util.function.Supplier;

public class SupplierExample {
    public static void main(String[] args) {
        Supplier<Double> randomValueSupplier = () -> Math.random();
        System.out.println("Random Value: " + randomValueSupplier.get());
    }
}
```

 `Supplier` 인터페이스는 값을 생성하고 반환하는데 사용되며, 위 예제에서는 `Math.random()`을 사용해 임의의 값을 반환한다.

## 3. Function

**Function**은 입력 값을 받아서 변환한 결과를 반환하는 함수형 인터페이스로 데이터의 변환, 매핑 등에 주로 사용된다.

**메소드 시그니처**:
```java
R apply(T t)
```

**예제**:
```java
import java.util.function.Function;

public class FunctionExample {
    public static void main(String[] args) {
        Function<String, Integer> stringLength = str -> str.length();
        int length = stringLength.apply("Hello, Java 8!");
        System.out.println("Length: " + length);
    }
}
```

 `Function` 인터페이스는 입력을 받아 변환한 결과를 반환하며, 이 예제에서는 문자열의 길이를 계산하는 함수이다.

## 4. Operator

**Operator**는 입력과 결과가 같은 타입일 때 사용되는 함수형 인터페이스로  `Function`의 일종이지만, 입력과 출력 타입이 동일한 경우 사용된다. 자바 8에서는 **UnaryOperator**와 **BinaryOperator**가 있다.

**UnaryOperator 메소드 시그니처**:
```java
T apply(T t)
```

**BinaryOperator 메소드 시그니처**:
```java
T apply(T t1, T t2)
```

**예제**:
```java
import java.util.function.BinaryOperator;

public class OperatorExample {
    public static void main(String[] args) {
        BinaryOperator<Integer> sum = (a, b) -> a + b;
        int result = sum.apply(10, 20);
        System.out.println("Sum: " + result);
    }
}
```

`BinaryOperator`는 두 개의 인수를 받아 같은 타입의 결과를 반환하며, 위 코드에서는 두 정수의 합을 계산한다.

## 5. Predicate

**Predicate**는 입력 값을 받아서 `boolean` 타입의 결과를 반환하는 함수형 인터페이스로 주로 조건 테스트나 필터링에 사용된다.

**메소드 시그니처**:
```java
boolean test(T t)
```

**예제**:
```java
import java.util.function.Predicate;

public class PredicateExample {
    public static void main(String[] args) {
        Predicate<String> isEmpty = str -> str.isEmpty();
        boolean result = isEmpty.test("");
        System.out.println("Is empty: " + result);
    }
}
```

`Predicate`는 조건을 평가하여 `boolean` 값을 반환하며, 위 예제에서는 문자열이 비어 있는지 여부를 확인하는 조건으로 사용했다.

## 요약

| 함수형 인터페이스      | 메소드 시그니처          | 활용                              | 매개변수 | **반환 값** |
| -------------- | ----------------- | ------------------------------- | ---- | -------- |
| Consumer<T>    | void accept(T t)  | 입력 값을 받아 소비하는 동작에 사용            | O    | X        |
| Supplier<T>    | T get()           | 값을 제공하는 동작에 사용                  | X    | O        |
| Function<T, R> | R apply(T t)      | 값을 변환하여 반환하는 동작에 사용             | O    | O        |
| Operator       | R apply(T t)      | 같은 타입의 값을 연산하고 반환하는 동작에 사용      | O    | O        |
| Predicate<T>   | boolean test(T t) | 조건을 평가하여 `boolean`을 반환하는 동작에 사용 | O    | O        |


이러한 인터페이스들은 람다 표현식과 함께 사용하여 코드를 간결하고 읽기 쉽게 만들어 준다. 

