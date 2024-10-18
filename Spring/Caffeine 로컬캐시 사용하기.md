---
noteID: cedf264b-a5a6-497e-af84-33df8f715f1b
---
# CaffeineCache를 이용한 로컬 캐시 설정하기

스프링부트에서 애플리케이션의 성능을 최적화하기 위해 캐시를 사용하는 방법 중 하나는 CaffeineCache를 활용하는 것이다. Caffeine은 높은 성능의 로컬 캐시 라이브러리로, 빠르고 다양한 기능을 제공해 애플리케이션 성능 향상에 도움을 준다. 이 글에서는 CaffeineCache를 스프링부트에서 로컬 캐시로 설정하는 방법을 예제로 살펴보겠다.

## 1. CaffeineCache 설정 준비
CaffeineCache를 사용하기 위해서는 의존성을 추가해야 한다. `build.gradle` 파일에 다음과 같은 의존성을 추가한다.

```groovy
implementation 'com.github.ben-manes.caffeine:caffeine:3.1.5'
```

이제 Caffeine을 사용할 준비가 되었다. 다음은 Caffeine을 로컬 캐시로 설정하는 방법을 알아보자.

## 2. CacheConfig 클래스 작성
캐시 설정을 활성화하기 위해 `@EnableCaching` 애너테이션을 추가한 `CacheConfig` 클래스를 작성한다. 이 클래스는 캐시 기능을 활성화하는 역할을 한다.

```java
@EnableCaching
@Configuration
public class CacheConfig {
    // 추가적인 캐시 설정이 필요한 경우 이곳에 작성할 수 있다.
}
```

`@EnableCaching` 애너테이션을 사용하여 애플리케이션의 캐시 기능을 활성화한다.

## 3. LocalCacheType Enum 정의
캐시 타입을 관리하기 위해 `LocalCacheType`이라는 Enum을 정의한다. 이 Enum은 캐시의 이름, 설명, 만료 시간, 최대 크기 등을 지정한다.

```java
@Getter
@RequiredArgsConstructor
public enum LocalCacheType {

    COUNTRY("국가", "country", 1, 1000);

    private final String description;
    private final String cacheName;
    private final int expiredAfterWrite;
    private final int maximumSize;
}
```

`LocalCacheType` Enum은 각 캐시 타입의 설정을 정의하고, 캐시의 이름, 만료 시간 및 최대 크기를 지정할 수 있다. 이 정보를 기반으로 Caffeine 캐시를 설정하게 된다.

## 4. LocalCacheConfig 클래스 작성
Caffeine 캐시를 실제로 생성하고 관리하기 위해 `LocalCacheConfig` 클래스를 작성한다. 이 클래스에서는 Caffeine 캐시 빈을 생성하고, 캐시 매니저를 설정한다.

```java
@Configuration
public class LocalCacheConfig {

    @Bean
    public List<CaffeineCache> caffeineCaches() {
        return Arrays.stream(LocalCacheType.values())
            .map(cache -> new CaffeineCache(cache.getCacheName(), Caffeine.newBuilder().recordStats()
                .expireAfterWrite(cache.getExpiredAfterWrite(), TimeUnit.HOURS)
                .maximumSize(cache.getMaximumSize())
                .build()))
            .toList();
    }

    @Bean
    public CacheManager localCacheManager(List<CaffeineCache> caffeineCaches) {
        SimpleCacheManager cacheManager = new SimpleCacheManager();
        cacheManager.setCaches(caffeineCaches);

        return cacheManager;
    }
}
```

- `caffeineCaches()` 메소드는 `LocalCacheType` Enum에 정의된 각 캐시 설정을 기반으로 Caffeine 캐시 인스턴스를 생성한다. `expireAfterWrite()`를 통해 캐시의 만료 시간을 설정하고, `maximumSize()`를 통해 캐시의 최대 크기를 설정한다.
- `localCacheManager()` 메소드는 `SimpleCacheManager`를 사용하여 Caffeine 캐시들을 관리하는 `CacheManager` 빈을 생성한다. 

## 5. CaffeineCache 사용하기
이제 CaffeineCache가 설정된 로컬 캐시를 사용할 준비가 되었다. 캐시를 사용하고자 하는 메소드에 `@Cacheable` 애너테이션을 추가하여 캐싱 기능을 적용할 수 있다.

```java
@Service
public class CountryService {

    @Cacheable(value = "country")
    public List<Country> getAllCountries() {
        // 데이터베이스에서 국가 정보를 가져오는 로직
        return countryRepository.findAll();
    }
}
```

위 코드에서는 `getAllCountries()` 메소드의 결과를 `country` 캐시에 저장한다. 최초 호출 시에는 데이터베이스에서 데이터를 가져오고, 이후 호출 시에는 캐시된 데이터를 반환하게 된다.

## 6. 결론
CaffeineCache를 스프링부트에서 로컬 캐시로 설정하는 것은 간단하면서도 강력한 성능 최적화 방법이다. 위에서 설명한 것처럼 `@EnableCaching`, `LocalCacheType` Enum, 그리고 `LocalCacheConfig` 클래스를 사용하여 Caffeine 캐시를 설정할 수 있다. 이를 통해 반복적인 데이터베이스 호출을 줄이고, 애플리케이션의 응답 시간을 크게 개선할 수 있다.
