# @Async 어노테이션으로 비동기 처리하는 방법

스프링 프레임워크는 간단한 비동기 처리를 위해 **@Async** 어노테이션을 제공한다. 이 어노테이션을 활용하면 비동기 메소드를 쉽게 정의하고 실행할 수 있어, 메인 스레드의 부하를 줄이거나, 긴 시간 동안 실행되는 작업을 별도의 스레드에서 수행할 수 있다. 

## 1. @Async 설정

비동기 처리를 사용하기 위해서는 먼저 스프링 애플리케이션에서 **@EnableAsync** 어노테이션을 통해 비동기 기능을 활성화해야 한다.

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;

@Configuration
@EnableAsync
public class AsyncConfig {
    
}
```

위 코드에서 `@EnableAsync` 어노테이션을 추가하여 비동기 기능을 활성화했는데 이를 통해 스프링은 `@Async` 어노테이션이 붙은 메소드를 비동기 방식으로 실행할 수 있게 된다.

## 2. @Async 사용법

`@Async` 어노테이션은 메소드 위에 붙여 비동기로 실행되도록 지정한다. 비동기 메소드는 일반적으로 **`void`** 또는 **`Future`** 타입의 반환값을 가지며 **`CompletableFuture`**와 같은 비동기 처리 객체도 사용할 수 있다.

```java
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.util.concurrent.CompletableFuture;

@Service
public class AsyncService {

    @Async
    public void asyncMethod() {
        System.out.println("비동기 작업 수행 중...");
        try {
            Thread.sleep(3000); // 3초 동안 지연
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("비동기 작업 완료");
    }

    @Async
    public CompletableFuture<String> asyncMethodWithReturn() {
        try {
            Thread.sleep(3000); // 3초 동안 지연
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return CompletableFuture.completedFuture("비동기 작업 결과");
    }
}
```

- **`asyncMethod()`**: 비동기 작업을 수행하지만 반환 값이 없으며 이 메소드는 메인 스레드와 별도로 실행
- **`asyncMethodWithReturn()`**: 비동기 작업을 수행하고 **`CompletableFuture`** 를 반환. 호출자는 이 객체를 통해 작업 결과를 받을 수 있음

## 3. @Async 사용 사례

### 백그라운드 작업 처리
`@Async` 어노테이션은 사용자 요청 처리 중 백그라운드에서 긴 시간 동안 실행되어야 하는 작업을 수행할 때 유용하다. 예를 들어, 이메일 발송, 파일 처리, 알림 발송 등 시간이 오래 걸릴 수 있는 작업을 비동기로 처리하여 메인 스레드가 다른 작업을 계속할 수 있도록 할 수있다.

```java
@Service
public class NotificationService {

    @Async
    public void sendEmailNotification(String email) {
        // 이메일 발송 로직
        System.out.println("이메일 발송 중: " + email);
    }
}
```

이렇게 하면 이메일 발송 작업은 별도의 스레드에서 처리되어 사용자에게 빠르게 응답을 반환할 수 있다.

### 대량 데이터 처리
대량의 데이터를 처리하는 작업을 비동기로 수행하면 시스템의 응답 속도를 개선할 수 있다. 예를 들어, 데이터베이스에 저장된 수많은 데이터를 처리하거나 외부 API를 호출해 데이터를 동기화하는 작업 등에 비동기 처리를 사용할 수 있다.

```java
@Service
public class DataProcessingService {

    @Async
    public void processLargeData() {
        // 대량 데이터 처리 로직
        System.out.println("대량 데이터 처리 중...");
    }
}
```

## 4. @Async 실행 시 주의사항

- **리턴 타입**: 비동기 메소드는 `void` 타입 또는 `Future`, `CompletableFuture`와 같은 타입으로 정의 하지 않으면 호출하는 측에서는 비동기 작업의 결과를 받을 수 없다.
- **Proxy를 통한 호출**: `@Async` 어노테이션이 적용된 메소드는 스프링이 프록시를 생성하여 비동기로 실행되므로 같은 클래스 내에서 메소드를 호출하면 비동기 처리가 되지 않는다. 이를 해결하려면 비동기 메소드를 다른 클래스에서 호출하거나, `ApplicationContext`를 이용해 빈을 가져와 호출해야 한다.
- **예외 처리**: 비동기 메소드에서 발생하는 예외는 기본적으로 메인 스레드에 영향을 주지 않으므로, `CompletableFuture.exceptionally()`와 같은 방법을 사용해 예외를 처리해야 한다.

## 5. 전체 코드 예제

```java
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
public class AsyncApplication {

    public static void main(String[] args) {
        SpringApplication.run(AsyncApplication.class, args);
    }

    @Bean
    CommandLineRunner run(AsyncService asyncService) {
        return args -> {
            asyncService.asyncMethod();
            asyncService.asyncMethodWithReturn().thenAccept(result -> {
                System.out.println("비동기 결과: " + result);
            });
            System.out.println("메인 스레드 작업 완료");
        };
    }
}
```

이 코드에서는 `CommandLineRunner`를 사용하여 애플리케이션 시작 시 비동기 메소드를 호출하고, 결과를 출력한다. (비동기 메소드가 실행되는 동안 메인 스레드는 별도의 작업을 계속해서 수행)

## 6. 마무리

스프링의 `@Async` 어노테이션은 비동기 작업을 간단하게 구현할 수 있게 해준다. 이를 통해 애플리케이션의 성능과 응답성을 높이고, 사용자 경험을 개선할 수 있다. 다만 `@Async`를 사용할 때는 프록시 패턴에 대한 이해가 선행되어야 한다.

