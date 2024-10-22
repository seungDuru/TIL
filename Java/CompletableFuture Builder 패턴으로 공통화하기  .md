# CompletableFuture Builder 패턴으로 공통화하기

```java
import java.util.concurrent.CompletableFuture;  
import java.util.concurrent.Executor;  
import java.util.concurrent.Executors;  
import java.util.function.Function;  
import java.util.function.Supplier;  
  
public class AsyncBuilder<T> {  
   
    private CompletableFuture<T> future;  
    private static final Executor executor = Executors.newFixedThreadPool(10);  
  
    private AsyncBuilder(Supplier<T> initialTask) {  
        this.future = CompletableFuture.supplyAsync(initialTask, executor);  
    }  
  
    public static <T> AsyncBuilder<T> startWith(Supplier<T> initialTask) {  
        return new AsyncBuilder<>(initialTask);  
    }  
  
    public <U> AsyncBuilder<U> thenApply(Function<T, U> nextTask) {  
        CompletableFuture<U> nextFuture = future.thenApplyAsync(nextTask, executor);  
        return new AsyncBuilder<>(() -> nextFuture.join());  
    }  
  
    public <U> AsyncBuilder<U> thenCompose(Function<T, CompletableFuture<U>> nextTask) {  
        CompletableFuture<U> nextFuture = future.thenComposeAsync(nextTask, executor);  
        return new AsyncBuilder<>(() -> nextFuture.join());  
    }  
  
    public AsyncBuilder<T> handleException(Function<Throwable, T> exceptionHandler) {  
        future = future.exceptionally(exceptionHandler);  
        return this;  
    }  
  
    public CompletableFuture<T> build() {  
        return future;  
    }  
  
    public T join() {  
        return future.join();  
    }  
}
```
  

## 사용예시

```java
import org.springframework.stereotype.Service;  
import java.util.concurrent.CompletableFuture;  
  
@Service  
public class MyService {  
  
    public String processTasks() {  
        return AsyncBuilder  
            .startWith(() -> {  
                // 첫 번째 비동기 작업  
                return "First Result";  
            })  
            .thenApply(firstResult -> {  
                // 첫 번째 결과를 이용한 두 번째 작업  
                return "Processed " + firstResult;  
            })  
            .thenCompose(processedResult -> {  
                // 두 번째 결과를 이용한 세 번째 비동기 작업  
                return CompletableFuture.supplyAsync(() -> {  
                    return processedResult + " and Composed Result";  
                });  
            })  
            .handleException(ex -> {  
                // 예외 발생 시 처리 로직  
                return "Error: " + ex.getMessage();  
            })  
            .join(); // 최종 결과를 반환  
    }  
}
```