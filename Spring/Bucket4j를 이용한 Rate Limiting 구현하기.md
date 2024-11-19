스프링 기반 백엔드 시스템에서 클라이언트 요청을 효율적으로 제어하고 서비스 남용을 방지하기 위해 **Rate Limiting**을 도입하는 것이 중요하다. 이러한 Rate Limiting을 간편하게 구현할 수 있는 라이브러리 중 하나가 바로 **Bucket4j** 이다.

## 1. Bucket4j란?

**Bucket4j**는 버킷 알고리즘을 기반으로 하는 Rate Limiting 라이브러리 이다. 버킷 알고리즘은 정해진 수의 토큰을 버킷에 넣고, 요청이 들어올 때마다 토큰을 소모하여 일정한 비율로 요청을 처리한다. 일정 시간마다 토큰이 다시 추가되기 때문에 과도한 요청이 들어오는 것을 제한할 수 있다.

Bucket4j는 스프링 부트와 쉽게 통합할 수 있으며, 네트워크 자원 남용을 방지하고 시스템 안정성을 높이는 데 유용하게 사용할 수 있다.

## 2. Bucket4j 의존성 추가하기

Gradle 환경에서 Bucket4j를 사용하기 위해서는 `build.gradle` 파일에 다음과 같이 의존성을 추가한다.

```groovy
dependencies {
    implementation 'com.github.vladimir-bukhtoyarov:bucket4j-core:7.4.0'
}
```

## 3. Bucket4j Rate Limiting 구현하기

다음은 Bucket4j를 활용하여 간단한 Rate Limiting을 구현하는 예제이다. 이 예제에서는 각 IP 주소마다 초당 5개의 요청을 허용하고, 초과하는 경우 요청을 차단하도록 설정한다.

### 3.1 Bean으로 Rate Limiting 설정

Bucket4j 설정을 스프링 빈으로 등록하여 여러 곳에서 재사용할 수 있도록 한다. 이를 통해 Rate Limiting 설정을 중앙에서 관리하고 코드 중복을 줄일 수 있다.

```java
import io.github.bucket4j.Bandwidth;
import io.github.bucket4j.Bucket;
import io.github.bucket4j.Bucket4j;
import io.github.bucket4j.Refill;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

@Configuration
public class Bucket4jConfiguration {

    @Bean
    public Bucket rateLimitBucket() {
        Bandwidth limit = Bandwidth.classic(5, Refill.greedy(5, Duration.ofSeconds(1)));
        return Bucket4j.builder()
                .addLimit(limit)
                .build();
    }
}
```

- **Bandwidth**: 버킷의 대역폭을 의미하며, 요청을 처리할 수 있는 토큰의 수와 토큰이 채워지는 속도를 정의합니다. `Bandwidth.classic(5, Refill.greedy(5, Duration.ofSeconds(1)))`에서 `5`는 최대 토큰 수를 의미하며, `Duration.ofSeconds(1)`은 매 1초마다 `5`개의 토큰을 채우는 것을 의미합니다.
    
- **Refill**: 버킷이 토큰을 채우는 방식입니다. `Refill.greedy(5, Duration.ofSeconds(1))`은 토큰이 고갈되었을 때 매 1초마다 `5`개의 토큰을 한 번에 채우는 방식을 의미합니다.

### 3.2 Rate Limiting 컨트롤러에 적용하기

```java
import io.github.bucket4j.Bucket;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestHeader;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class RateLimitingController {

    private final Bucket bucket;

    public RateLimitingController(Bucket bucket) {
        this.bucket = bucket;
    }

    @GetMapping("/api/limited")
    public String limitedEndpoint(@RequestHeader(value = "X-Forwarded-For", required = false) String ipAddress) {
        if (bucket.tryConsume(1)) {
            return "Request accepted";
        } else {
            return "Too many requests - please try again later.";
        }
    }
}
```

### 3.3 코드 설명

- **Bucket 빈 주입**: `RateLimitingController`에서 `Bucket`을 생성자 주입 방식으로 받아와 사용한다. 이를 통해 설정을 중앙에서 관리하고, 여러 컨트롤러나 서비스에서도 동일한 Rate Limiting 설정을 공유할 수 있다.
    
- **tryConsume()**: 버킷에서 토큰을 소비하고, 토큰이 남아 있으면 요청을 허용하고 그렇지 않으면 요청을 차단한다.

## 4. Rate Limiting의 중요성

### 4.1 시스템 보호

Rate Limiting을 통해 서비스 남용으로 인한 서버 과부하를 방지할 수 있다. 이를 통해 안정적인 시스템 운영이 가능하며, 다른 사용자가 공평하게 리소스를 사용할 수 있도록 한다.

### 4.2 DDoS 공격 방어

Rate Limiting은 DDoS(Distributed Denial of Service) 공격에 대한 기본적인 방어 수단으로 활용될 수 있다. 요청 빈도를 제한함으로써 악의적인 대량 요청이 서버에 무리를 주는 것을 방지할 수 있다.

## 5. 고려사항

### 5.1 버킷 설정 최적화

Rate Limiting을 적용할 때는 각 애플리케이션의 특성에 맞는 적절한 제한값을 설정하는 것이 중요하다. 너무 빡빡한 제한은 정당한 사용자의 불편을 초래할 수 있으며, 너무 느슨한 제한은 Rate Limiting의 효과를 떨어뜨릴 수 있다.

### 5.2 클러스터 환경에서의 동기화

위 예제는 단일 서버 환경을 기준으로 한 것으로 만약 클러스터 환경에서 Rate Limiting을 적용하려면, Redis와 같은 중앙 집중식 스토리지를 사용하여 모든 서버 간의 Rate Limiting 상태를 공유하는 것이 필요하다.
