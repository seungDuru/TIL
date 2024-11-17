# Request 타임아웃

스프링 기반 백엔드 시스템에서 요청 처리의 효율성을 높이기 위해서는 타임아웃 설정이 매우 중요하다. 요청이 일정 시간 내에 완료되지 않을 경우 시스템의 안정성을 저해하고 사용자 경험에 악영향을 줄 수 있다. 

## 1. Request 타임아웃이란?

Request 타임아웃은 서버 또는 클라이언트가 특정 요청을 수행하는 데 걸리는 시간을 제한하는 설정이다. 요청이 설정된 시간 내에 완료되지 않으면 해당 요청은 타임아웃으로 간주되어 실패하게 된다. 이는 서버가 과부하에 걸리는 것을 방지하고, 자원을 효율적으로 사용하기 위한 핵심 요소다.

스프링에서의 타임아웃 설정은 다음과 같이 여러 단계에서 설정될 수 있다.

1. **연결 타임아웃(Connection Timeout)**: 서버와의 연결이 성립되는 데까지 걸리는 시간의 제한. 연결 타임아웃은 클라이언트가 서버에 연결을 시도할 때, 일정 시간 내에 연결이 성립되지 않으면 타임아웃으로 간주하고 실패 처리를 한다. 이는 주로 네트워크 지연이나 서버의 응답 불능 상태로 인해 연결이 지연되는 상황에서 사용된다. 예를 들어, 클라이언트가 서버에 요청을 보낼 때 서버가 너무 바쁘거나 네트워크가 불안정하여 연결이 성립되지 않는 경우, 연결 타임아웃을 통해 불필요한 대기를 방지할 수 있다.

2. **읽기 타임아웃(Read Timeout)**: 요청이 성공적으로 전송된 후 응답을 읽기 시작하는 데 걸리는 시간의 제한. 읽기 타임아웃은 클라이언트가 서버로부터 응답을 받기 시작하는 데까지 걸리는 시간을 제한한다. 요청이 서버에 도달한 후, 서버에서 처리된 응답을 가져오는 과정에서 시간이 너무 오래 걸리는 경우 타임아웃이 발생한다. 이는 서버가 복잡한 작업을 수행하거나 데이터베이스에서 데이터를 읽는 데 시간이 오래 걸릴 때 유용하다. 읽기 타임아웃을 설정함으로써 응답이 너무 지연될 경우, 클라이언트는 이를 중단하고 대체 로직을 수행할 수 있다.

3. **쓰기 타임아웃(Write Timeout)**: 클라이언트가 데이터를 서버로 전송하는 데 걸리는 시간의 제한. 쓰기 타임아웃은 클라이언트가 요청 데이터를 서버로 전송하는 동안 일정 시간이 초과될 경우 발생한다. 예를 들어, 업로드할 파일이 크거나 서버와의 네트워크 상태가 불안정한 경우, 데이터를 전송하는 데 시간이 오래 걸릴 수 있다. 이러한 상황에서 쓰기 타임아웃을 설정하면 클라이언트는 데이터를 전송하는 데 너무 많은 시간을 소비하지 않도록 제어할 수 있다.

이들 타임아웃을 적절히 설정하면 네트워크 문제, 서버 과부하 등을 효과적으로 대응할 수 있다.

## 2. WebClient에서 타임아웃 설정하기

스프링 WebClient는 비동기 방식의 HTTP 요청을 처리하는 강력한 도구다. WebClient에서 타임아웃 설정을 통해 네트워크 지연이나 서버 문제를 보다 효과적으로 관리할 수 있다. 아래는 WebClient에서 연결, 읽기, 쓰기 타임아웃을 설정하는 방법에 대한 예제다. 

```java
import org.springframework.web.reactive.function.client.WebClient;
import reactor.netty.http.client.HttpClient;
import java.time.Duration;

public class WebClientTimeoutExample {

    public static void main(String[] args) {
        HttpClient httpClient = HttpClient.create()
                .responseTimeout(Duration.ofSeconds(5)) // 읽기 타임아웃 설정
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000); // 연결 타임아웃 설정

        WebClient webClient = WebClient.builder()
                .clientConnector(new ReactorClientHttpConnector(httpClient))
                .baseUrl("https://api.example.com")
                .build();

        webClient.get()
                .uri("/data")
                .retrieve()
                .bodyToMono(String.class)
                .timeout(Duration.ofSeconds(3)) // 요청 자체에 대한 타임아웃 설정
                .doOnError(throwable -> System.err.println("Request timed out: " + throwable.getMessage()))
                .subscribe(response -> System.out.println("Response: " + response));
    }
}
```

### 코드 설명

- **HttpClient 설정**: `HttpClient.create()`를 통해 연결 및 읽기 타임아웃을 설정한다.

  - `responseTimeout(Duration.ofSeconds(5))`: 서버의 응답을 기다리는 최대 시간을 5초로 설정한다.
  - `ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000`: 서버와의 연결이 3초 내에 이루어지지 않으면 타임아웃이 발생한다.

- **WebClient 빌더 사용**: `WebClient.builder()`를 통해 `ReactorClientHttpConnector`로 위에서 설정한 `HttpClient`를 연결한다.

- **요청 타임아웃 설정**: `.timeout(Duration.ofSeconds(3))`을 통해 전체 요청 시간이 3초를 초과할 경우 타임아웃을 발생시킨다.

## 3. 타임아웃 설정의 중요성

### 3.1 서버 안정성 향상

타임아웃 설정을 통해 서버가 오랫동안 응답하지 않는 요청을 처리하려고 대기하지 않도록 방지할 수 있다. 이는 서버 리소스를 확보하고 다른 요청들이 원활하게 처리될 수 있도록 한다.

### 3.2 사용자 경험 개선

사용자가 웹 애플리케이션에서 요청을 보내고 긴 시간 동안 응답을 기다려야 한다면 사용자는 불만을 느낄 수 있다. 타임아웃 설정을 통해 적절한 시점에 에러 메시지를 제공함으로써 사용자는 상황을 이해하고 다음 행동을 할 수 있게 된다.

## 4. 타임아웃 관련 고려사항

### 4.1 적절한 타임아웃 값 설정하기

너무 짧은 타임아웃은 네트워크 환경이 불안정한 상황에서 요청 실패를 자주 초래할 수 있으며, 너무 긴 타임아웃은 시스템에 과부하를 초래할 수 있다. 애플리케이션의 특성에 맞는 적절한 타임아웃 값을 설정하는 것이 중요하다.

### 4.2 전체 시스템의 타임아웃 일관성

서로 다른 서비스 간에 타임아웃 값이 일관되지 않으면 전체 시스템의 신뢰성이 떨어질 수 있다. 예를 들어, 프록시 서버와 백엔드 서버 간의 타임아웃 값이 상이하면 요청이 불필요하게 중단되거나 시스템 장애를 초래할 수 있다. 따라서 각 컴포넌트 간 타임아웃 값을 일관성 있게 맞추는 것이 좋다.

## 마무리

스프링에서 Request 타임아웃을 적절히 설정하는 것은 서버의 안정성과 사용자 경험 모두를 향상시키는 중요한 요소이다. WebClient를 사용할 때 연결, 읽기, 쓰기 타임아웃을 적절히 설정하여 보다 견고하고 신뢰성 있는 애플리케이션을 구축할 수 있다. 특히, 타임아웃을 잘못 설정하면 오히려 시스템 장애나 사용자의 불편함을 초래할 수 있으므로, 각 상황에 맞는 적절한 값 설정이 필수적이다.

