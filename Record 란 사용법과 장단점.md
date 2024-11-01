# Java Record: 설명, 사용법, 장단점

자바 14부터 미리보기(Preview) 기능으로 소개된 `Record`는 자바 16에서 정식 기능으로 추가되었다. Record는 불변 데이터 객체를 간결하게 표현하기 위해 고안된 새로운 클래스 유형이다. 

## 1. Java Record란 무엇인가?

Java Record는 데이터를 표현하기 위한 간결하고 불변(immutable)한 클래스를 정의하는 새로운 방식으로 일반적으로 데이터 객체를 정의할 때 필요한 필드, 생성자, `equals()`, `hashCode()`, `toString()` 메서드를 자동으로 생성해 주기 때문에, 코드 작성이 매우 간결해진다.

기존의 클래스를 정의할 때는 많은 보일러플레이트 코드가 필요했지만, Record를 사용하면 이러한 코드를 줄일 수 있다. Record는 값 중심의 데이터 구조를 정의하는 데 매우 유용하다.

### 예시: 간단한 Record 정의

```java
public record Person(String name, int age) {}
```
위의 코드에서 `Person`은 `name`과 `age`를 가지는 Record 인데 자바 컴파일러는 자동으로 생성자, 접근자(getter), `equals()`, `hashCode()`, `toString()` 메서드를 생성해준다.

## 2. Java Record의 장단점

### 장점

1. **보일러플레이트 코드 감소**: Record는 생성자, `equals()`, `hashCode()`, `toString()` 등을 자동으로 생성하기 때문에 코드가 간결해진다. 데이터 중심의 객체를 정의할 때 매우 편리하다.

2. **불변성 보장**: Record는 기본적으로 모든 필드가 `final`이며 불변이다. 이를 통해 불변 객체를 쉽게 만들 수 있어, 동시성 문제를 줄일 수 있다.

3. **간단한 데이터 모델 표현**: 간단한 데이터 모델(예: DTO, 값 객체)을 표현할 때 유용하다. 복잡한 로직이 없는 데이터 전송 객체에 적합하다.

### 단점

1. **유연성 부족**: Record는 필드가 `final`로 고정되어 있기 때문에, 필드를 변경할 수 없다. 즉, 가변 객체가 필요하다면 Record를 사용할 수 없다.

2. **상속 불가**: 모든 Record는 암묵적으로 `java.lang.Record` 클래스를 상속하기 때문에 다른 클래스를 상속할 수 없다. 따라서 상속을 통한 다형성이 필요하다면 Record를 사용할 수 없다.

3. **복잡한 로직 구현의 한계**: Record는 간단한 데이터 구조를 표현하는 데 적합하지만, 복잡한 로직이나 비즈니스 로직을 포함하는 객체로 사용하기에는 한계가 있다.

## 3. Record의 활용 예시

### 예제 코드: Record 사용

```java
public class RecordExample {
    public static void main(String[] args) {
        Person person = new Person("Alice", 30);
        System.out.println("Person 정보: " + person);
        System.out.println("이름: " + person.name());
        System.out.println("나이: " + person.age());
    }
}
```
위의 코드에서 `Person` Record를 생성하고, 그 정보를 출력한다. Record의 불변성과 자동으로 생성되는 메서드 덕분에 코드를 간결하게 유지할 수 있다.

## 마무리

Java Record는 불변 데이터 객체를 간결하게 표현할 수 있는 강력한 도구로, 보일러플레이트 코드를 줄이고 가독성을 높이는 데 좋으며 특히 값 중심의 객체를 다루기 좋으며, 불변성을 보장하여 동시성 문제를 줄일 수 있다. 그러나 Record는 상속을 지원하지 않고, 필드 변경이 불가능하기 때문에 가변 데이터 구조나 상속을 통한 다형성을 요구하는 경우에는 적합하지 않을 수 있다.

