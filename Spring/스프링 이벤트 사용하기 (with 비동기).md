# 스프링 이벤트

스프링에서 이벤트 기반 프로그래밍은 애플리케이션의 결합도를 줄이고 비동기 처리를 손쉽게 구현할 수 있는 매우 유용한 기법이다. 스프링 이벤트는 기본적으로 싱크로나이즈드(Synchronized) 방식으로 동작하지만, 비동기 처리가 필요한 경우 간단한 설정만으로 비동기 이벤트로 변환할 수 있다. 

## 1. 스프링 이벤트란?

스프링 이벤트는 애플리케이션의 한 부분에서 발생한 이벤트를 다른 부분에서 처리하도록 함으로써 모듈 간의 결합도를 낮추는 데 기여한다. 스프링 프레임워크는 이러한 이벤트를 발행하고 수신하는 기능을 내장하고 있어, 개발자가 필요한 이벤트를 쉽게 정의하고 처리할 수 있다.

### 1.1 이벤트 구성 요소
- **이벤트 클래스**: 이벤트 객체로 사용하기 위해서는 `ApplicationEvent` 클래스를 상속받거나 단순히 일반 클래스를 정의할 수 있다.
- **이벤트 발행자(Publisher)**: 특정 작업이 발생할 때 이벤트를 발행하는 역할을 수행한다. `ApplicationEventPublisher` 인터페이스를 통해 구현할 수 있다.
- **이벤트 리스너(Listener)

스프링 이벤트 리스너는 발행된 이벤트를 수신하고 처리하는 역할을 한다. 상황에 따라 여러 종류의 리스너를 사용할 수 있다.

### 리스너 종류
1. **동기 리스너**: 기본적으로 스프링 이벤트 리스너는 동기적으로 동작한다. 이벤트가 발행되면 리스너는 해당 이벤트를 처리할 때까지 다른 작업을 수행하지 않는다. 이는 중요한 처리가 즉시 이루어져야 하거나 순차적인 처리가 필요한 경우 적합하다.

2. **비동기 리스너**: `@Async` 어노테이션을 사용하여 비동기 리스너로 설정할 수 있다. 비동기 리스너는 이벤트가 발행되었을 때 다른 작업과 병렬로 처리되며, 특히 응답 속도를 중요시하는 경우에 유용하다.

3. **트랜잭션 이벤트 리스너**: `@TransactionalEventListener`를 사용하여 트랜잭션 상태에 따라 이벤트를 처리할 수 있다. 예를 들어, 트랜잭션이 성공적으로 완료된 후에만 특정 이벤트를 처리하거나, 트랜잭션 롤백 후에 이벤트를 처리하도록 설정할 수 있다. 이를 통해 데이터의 일관성을 유지하면서 이벤트를 처리할 수 있다.

### 리스너 사용 시 고려사항
- **예외 처리**: 이벤트 리스너 내부에서 발생하는 예외는 기본적으로 상위 호출자에게 전파되지 않기 때문에, 리스너 내부에서 예외를 직접 처리하거나 로깅하는 것이 필요하다.
- **성능**: 동기 리스너는 이벤트가 처리될 때까지 다른 작업을 막기 때문에, 리스너 내에서 시간이 많이 걸리는 작업이 있을 경우 성능에 영향을 줄 수 있다. 이런 경우 비동기 리스너를 사용하는 것이 더 적합하다.
- **다중 리스너**: 하나의 이벤트에 대해 여러 개의 리스너가 존재할 수 있으며, 이 경우 모든 리스너가 해당 이벤트를 처리한다. 이벤트 처리의 순서가 중요하다면 적절한 우선순위 설정이 필요하다.**: 발행된 이벤트를 수신하고 처리하는 역할을 한다. `@EventListener` 어노테이션을 사용해 간단히 구현할 수 있다.

```java
public class UserRegisteredEvent {
    private final String username;

    public UserRegisteredEvent(String username) {
        this.username = username;
    }

    public String getUsername() {
        return username;
    }
}

@Component
public class UserService {
    private final ApplicationEventPublisher eventPublisher;

    public UserService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    public void registerUser(String username) {
        // 사용자 등록 로직
        eventPublisher.publishEvent(new UserRegisteredEvent(username));
    }
}

@Component
public class WelcomeEmailListener {
    @EventListener
    public void handleUserRegisteredEvent(UserRegisteredEvent event) {
        System.out.println("Sending welcome email to " + event.getUsername());
    }
}
```
위 코드에서 `UserService`는 사용자 등록 후 `UserRegisteredEvent`를 발행하고, `WelcomeEmailListener`는 해당 이벤트를 수신하여 환영 이메일을 보낸다.

## 2. 비동기 이벤트 처리

스프링 이벤트는 기본적으로 동기적으로 처리된다. 즉, 이벤트가 발행되면 리스너가 해당 이벤트를 처리할 때까지 발행자는 기다리게 된다. 하지만, 비동기적으로 이벤트를 처리하고자 하는 경우가 더 효율적일 수 있다. 예를 들어, 사용자 등록 후 이메일을 보내는 작업은 비동기적으로 처리하여 사용자 등록의 응답 시간을 줄일 수 있다.

### 2.1 `@Async` 어노테이션 사용
비동기 이벤트 처리는 리스너 메소드에 `@Async` 어노테이션을 추가함으로써 간단히 구현할 수 있다. 이를 위해서는 별도의 스프링 비동기 설정이 필요하다.

```java
@Configuration
@EnableAsync
public class AsyncConfig {
}

@Component
public class WelcomeEmailListener {
    @EventListener
    @Async
    public void handleUserRegisteredEvent(UserRegisteredEvent event) {
        System.out.println("Sending welcome email to " + event.getUsername());
    }
}
```
위 코드에서는 `WelcomeEmailListener`가 비동기적으로 실행되도록 설정하였다. 이를 통해 사용자 등록 과정에서 이메일 전송이 독립적으로 수행되어 전체 응답 시간을 줄일 수 있다.

### 2.2 비동기 처리의 주의사항
- **트랜잭션 전파**: 비동기 이벤트 리스너는 원래의 트랜잭션과 별도로 실행되기 때문에, 원래 트랜잭션이 완료되지 않은 상태에서 이벤트가 발행될 수 있다. 이는 데이터의 일관성 문제를 일으킬 수 있으므로, 필요한 경우 트랜잭션을 명시적으로 관리해야 한다.
- **예외 처리**: 비동기 이벤트 처리 중 발생하는 예외는 원래의 호출자에게 전달되지 않는다. 따라서, 이벤트 리스너 내부에서 예외를 적절히 처리해야 한다.

## 3. 트랜잭션과 이벤트

스프링 이벤트와 트랜잭션은 밀접한 관계가 있을 수 있다. 예를 들어, 트랜잭션이 완료된 후에만 특정 이벤트를 발행하고 싶다면, `TransactionPhase` 옵션을 활용하여 이를 제어할 수 있다.

### 3.1 트랜잭션 후 이벤트 발행
스프링은 `@TransactionalEventListener` 어노테이션을 사용하여 트랜잭션 상태에 따라 이벤트를 발행하도록 할 수 있다.

```java
@Component
public class OrderService {
    @Transactional
    public void placeOrder(Order order) {
        // 주문 처리 로직
        eventPublisher.publishEvent(new OrderPlacedEvent(order));
    }
}

@Component
public class OrderListener {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleOrderPlacedEvent(OrderPlacedEvent event) {
        System.out.println("Order " + event.getOrder().getId() + " has been placed successfully.");
    }
}
```
위 코드에서 `OrderListener`는 트랜잭션이 성공적으로 커밋된 후에만 이벤트를 처리하도록 설정되어 있다. 이를 통해 트랜잭션이 롤백될 경우 이벤트가 발행되지 않도록 보장할 수 있다.

### 3.2 `TransactionPhase` 옵션
- **AFTER_COMMIT**: 트랜잭션 커밋 후 이벤트 처리
- **AFTER_ROLLBACK**: 트랜잭션 롤백 후 이벤트 처리
- **AFTER_COMPLETION**: 트랜잭션 완료 후(커밋 또는 롤백) 이벤트 처리
- **BEFORE_COMMIT**: 트랜잭션 커밋 직전에 이벤트 처리

이 옵션을 적절히 사용하면, 트랜잭션의 상태에 따라 유연하게 이벤트를 처리할 수 있다.

## 4. 이벤트 사용 시의 고려사항

### 4.1 이벤트 남용 주의
이벤트를 지나치게 많이 사용하면, 시스템의 흐름을 파악하기 어렵게 만들 수 있으며, 디버깅과 유지보수가 힘들어질 수 있다. 이벤트는 모듈 간의 결합도를 낮추는 장점이 있지만, 지나치게 복잡한 이벤트 체인은 애플리케이션의 복잡성을 증가시킬 수 있다.

### 4.2 이벤트 순서 보장
스프링 이벤트는 기본적으로 순서가 보장되지 않는다. 따라서 이벤트 처리 순서가 중요하다면, 이를 별도로 관리해야 한다. 예를 들어, 특정 이벤트는 반드시 다른 이벤트 후에 처리되어야 하는 경우에는 이벤트 체인을 설계할 때 주의가 필요하다.

## 마무리
스프링 이벤트 시스템은 모듈 간의 결합도를 낮추고 비동기 처리를 통해 애플리케이션의 성능을 향상시키는 데 큰 도움이 된다. 비동기 처리와 트랜잭션 연계 기능을 활용하면, 더 유연하고 효율적인 애플리케이션 설계가 가능하다. 하지만, 이벤트를 남발하거나 지나치게 복잡하게 설계하는 것은 피해야 하며, 항상 적절한 수준에서 활용할 필요가 있다.

스프링 이벤트를 사용해 모듈 간의 유연성을 극대화하면서도, 트랜잭션과 비동기 처리의 복잡성을 잘 다루는 것이 성공적인 애플리케이션 개발의 열쇠가 될 것이다.

