스프링 환경에서 외부 시스템과의 통신 중 발생할 수 있는 오류를 유연하게 처리하고, 애플리케이션의 안정성을 높이기 위해 Resilience4j를 많이 사용한다. 

## 1. Resilience4j란?

Resilience4j는 현대 분산 시스템에서 필수적인 탄력성을 구현하기 위한 경량의 라이브러리로 Netflix의 Hystrix가 더 이상 유지보수되지 않는 이후로, 많은 개발자들이 Resilience4j를 대안으로 선택하고 있다. 주요 기능으로는 다음과 같다.

- **서킷 브레이커 (Circuit Breaker)**: 오류가 반복되는 경우 외부 시스템과의 연결을 차단하여 전체 시스템이 영향을 받지 않도록 보호
- **리트라이 (Retry)**: 실패한 요청을 일정 횟수까지 재시도 처리
- **레이트 리미터 (Rate Limiter)**: 특정 시간 동안의 요청 횟수를 제한하여 서비스 오버로드를 방지
- **벌크헤드 (Bulkhead)**: 리소스를 구획화하여 하나의 서비스 실패가 다른 서비스에 영향을 미치지 않도록 처리

이러한 기능을 통해 Resilience4j는 분산 시스템에서 안정적인 서비스를 제공할 수 있도록 돕는다.

## 2. Resilience4j 사용 방법

자바 스프링부트에서 Resilience4j를 사용하려면 먼저 Gradle에 Resilience4j 의존성을 추가해야 한다.

### 2.1 Gradle 의존성 추가하기

먼저 `build.gradle` 파일에 Resilience4j와 스프링부트 스타터 의존성을 추가한다.

```groovy
implementation 'io.github.resilience4j:resilience4j-spring-boot3:2.0.2'
implementation 'org.springframework.boot:spring-boot-starter-aop'
```

AOP(Aspect Oriented Programming) 의존성을 추가하는 이유는 Resilience4j의 어노테이션 기반 구성을 사용하기 위함이다.

### 2.2 서킷 브레이커 설정하기

서킷 브레이커는 Resilience4j에서 가장 많이 사용되는 기능 중 하나로 외부 서비스가 지속적으로 실패하는 경우 서킷 브레이커를 통해 요청을 빠르게 차단함으로써 전체 애플리케이션의 성능 저하를 방지할 수 있다.

```java
@RestController
public class DemoController {

    private final DemoService demoService;

    public DemoController(DemoService demoService) {
        this.demoService = demoService;
    }

    @GetMapping("/demo")
    @CircuitBreaker(name = "demoService", fallbackMethod = "fallbackDemo")
    public String demoEndpoint() {
        return demoService.getData();
    }

    public String fallbackDemo(Throwable t) {
        return "Fallback response due to: " + t.getMessage();
    }
}
```

위 코드에서는 `@CircuitBreaker` 어노테이션을 사용하여 서킷 브레이커를 적용하였다  `fallbackMethod` 속성을 통해 실패 시 호출될 대체 메서드를 지정할 수 있다.

### 2.3 설정 파일 작성하기

서킷 브레이커의 동작 방식은 `application.yml` 파일에서 설정할 수 있다.

```yaml
resilience4j:
  circuitbreaker:
    instances:
      demoService:
        register-health-indicator: true
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
```

여기서 `sliding-window-size`는 서킷 브레이커가 상태를 평가하기 위해 고려하는 호출 횟수를 나타내며, `failure-rate-threshold`는 서킷을 열지 여부를 결정하기 위한 실패 비율이다.

### 2.4 리트라이와 레이트 리미터 사용하기

리트라이와 레이트 리미터를 적용하는 것도 비슷한 방식으로 가능하다. 예를 들어, 특정 API 호출이 실패할 경우 재시도하도록 하려면 `@Retry` 어노테이션을 사용할 수 있습니다.

```java
@Retry(name = "demoService", fallbackMethod = "fallbackDemo")
public String demoEndpoint() {
    return demoService.getData();
}
```

레이트 리미터는 `@RateLimiter` 어노테이션으로 적용할 수 있으며, 특정 API에 대한 호출 빈도를 제어할 수 있습니다.

## 3. Resilience4j 사용 시 주의사항

### 3.1 과도한 재시도와 서킷 브레이커 오픈

리트라이나 서킷 브레이커를 사용할 때는 설정 값을 신중하게 조정해야 합니다. 과도한 재시도는 서버에 불필요한 부하를 유발할 수 있으며, 서킷 브레이커가 자주 열리고 닫히는 경우에는 외부 서비스가 복구되었음에도 정상적인 호출이 어려울 수 있습니다. 따라서 적절한 `waitDuration`과 `retryAttempts` 설정이 중요합니다.

### 3.2 설정 복잡성 관리

Resilience4j는 다양한 기능을 제공하며, 설정할 수 있는 값도 많습니다. 이러한 설정을 잘못 관리하면 오히려 시스템의 복잡성이 증가하여 디버깅이 어려워질 수 있습니다. 따라서 설정 파일을 체계적으로 관리하고, 각 기능의 의미와 설정 값을 정확히 이해하고 적용해야 합니다.

### 3.3 외부 시스템 의존도 줄이기

Resilience4j는 외부 시스템의 장애로부터 애플리케이션을 보호하기 위한 도구입니다. 그러나 외부 시스템에 대한 의존도를 줄이는 것도 중요합니다. 서킷 브레이커나 리트라이가 빈번히 동작한다면 외부 시스템과의 의존도를 줄이기 위해 캐시나 대체 데이터 소스의 사용을 고려해야 합니다.

## 마무리

Resilience4j는 마이크로서비스와 분산 시스템 환경에서 서비스의 안정성과 탄력성을 높이는 데 매우 유용한 라이브러리입니다. 서킷 브레이커, 리트라이, 레이트 리미터 등의 기능을 통해 외부 시스템과의 통신에서 발생할 수 있는 문제를 효과적으로 관리할 수 있습니다. 다만 각 기능을 적절히 조합하고 설정 값을 신중히 관리하는 것이 중요합니다. 이를 통해 서비스 장애를 예방하고, 더 견고한 애플리케이션을 구축할 수 있을 것입니다.

