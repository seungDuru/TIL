# Redis를 이용한 스프링부트 글로벌 캐시 설정하기

스프링부트에서 애플리케이션 성능을 최적화하기 위해 글로벌 캐시로 Redis를 사용하고, 보조적으로 로컬 캐시로 Caffeine을 사용하는 방법에 대해 알아보자. Redis는 분산 환경에서의 데이터 공유와 글로벌 캐시 용도로 적합하며, 이를 통해 캐시 매니저 충돌 문제를 해결하고 각 서비스 메소드에서 필요한 캐시 매니저를 선택하여 사용할 수 있다. 이 글에서는 Redis를 글로벌 캐시로 사용하는 방법에 초점을 맞추고, Caffeine과 함께 설정하는 과정을 설명한다.

## 1. Redis 글로벌 캐시
글로벌 캐시는 분산 환경에서 데이터를 공유할 수 있도록 해주며, Redis는 대표적인 인메모리 데이터 스토어로 글로벌 캐시용으로 적합하다.

## 2. GlobalCacheType Enum 정의
Redis 글로벌 캐시 타입을 관리하기 위해 `GlobalCacheType`이라는 Enum을 정의한다. 이 Enum은 캐시의 이름, 설명, 만료 시간(TTL)을 지정한다.

### 코드: GlobalCacheType Enum 정의
```java
@Getter
@RequiredArgsConstructor
public enum GlobalCacheType {
    ALL_TRANSLATIONS_BY_LOCALE("번역", "translations", Duration.ofHours(1));

    private final String description;
    private final String cacheName;
    private final Duration ttl;
}
```

`GlobalCacheType` Enum은 글로벌 캐시의 설정을 정의하고, 각 캐시의 만료 시간 등을 지정할 수 있다.

## 3. Redis 글로벌 캐시 설정
글로벌 캐시 설정을 위해 Redis를 사용하여 `GlobalCacheConfig` 클래스를 작성한다. 이 클래스는 Redis 캐시 매니저를 빈으로 등록한다.

### 코드: Redis 글로벌 캐시 설정
```java
@Configuration
public class GlobalCacheConfig {

    private final RedisConnectionFactory redisConnectionFactory;

    public GlobalCacheConfig(RedisConnectionFactory redisConnectionFactory) {
        this.redisConnectionFactory = redisConnectionFactory;
    }

    @Bean
    public RedisCacheManager globalCacheManager() {
        GenericJackson2JsonRedisSerializer serializer = new GenericJackson2JsonRedisSerializer(objectMapper());

        Map<String, RedisCacheConfiguration> cacheConfigurations = EnumSet.allOf(GlobalCacheType.class).stream()
            .collect(Collectors.toMap(
                GlobalCacheType::getCacheName,
                cacheType -> RedisCacheConfiguration.defaultCacheConfig()
                    .entryTtl(cacheType.getTtl())
                    .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                    .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(serializer))
            ));

        return RedisCacheManager.builder(redisConnectionFactory)
            .withInitialCacheConfigurations(cacheConfigurations)
            .build();
    }

    private ObjectMapper objectMapper() {
        PolymorphicTypeValidator typeValidator = BasicPolymorphicTypeValidator.builder()
            .allowIfSubType(Object.class)
            .build();

        return new ObjectMapper()
            .enable(JsonGenerator.Feature.IGNORE_UNKNOWN)
            .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)
            .registerModule(new JavaTimeModule())
            .activateDefaultTyping(typeValidator, ObjectMapper.DefaultTyping.NON_FINAL, JsonTypeInfo.As.PROPERTY);
    }
}

```

### GenericJackson2JsonRedisSerializer의 장점
`GenericJackson2JsonRedisSerializer`는 Redis에 저장되는 데이터를 JSON 형식으로 직렬화하는 데 사용된다. 이를 사용하면 다음과 같은 장점이 있다:

1. **유연한 직렬화**: 다양한 객체 타입을 JSON 형식으로 직렬화할 수 있어 객체 타입에 대한 제약이 적다.
2. **가독성**: JSON은 가독성이 좋아, 디버깅이나 모니터링 시 데이터 구조를 쉽게 이해할 수 있다.
3. **타입 정보 보존**: `activateDefaultTyping`을 통해 객체의 타입 정보를 보존할 수 있어, 역직렬화 시 원래 객체 타입을 유지할 수 있다.

이러한 장점으로 인해 Redis에서 다양한 데이터를 쉽게 캐시하고 관리할 수 있다.  

### ObjectMapper 커스터마이징의 이유
`ObjectMapper`를 커스터마이징하는 이유는 Redis에 저장되는 데이터의 직렬화와 역직렬화 과정에서 유연성과 호환성을 제공하기 위해서이다. `objectMapper()` 메소드에서 다음과 같은 설정을 통해 커스터마이징을 한다:

1. **타입 정보 보존 (`activateDefaultTyping`)**: 객체를 JSON으로 직렬화할 때 타입 정보를 포함시켜, 역직렬화 시 원래 객체 타입을 유지할 수 있도록 한다. 이는 다양한 객체 타입을 Redis에 저장하고 관리할 때 매우 유용하다.
2. **알 수 없는 속성 무시 (`FAIL_ON_UNKNOWN_PROPERTIES`)**: 직렬화된 JSON에 예상치 못한 속성이 있을 경우 오류를 발생시키지 않고 무시하도록 설정하여, 데이터 구조의 변경에도 유연하게 대응할 수 있다.
3. **JavaTimeModule 등록 (`JavaTimeModule`)**: Java 8의 날짜와 시간 관련 객체(LocalDateTime 등)를 처리할 수 있도록 `JavaTimeModule`을 등록하여 직렬화/역직렬화의 호환성을 제공한다.
4. **알 수 없는 속성 무시 (`IGNORE_UNKNOWN`)**: 역직렬화 과정에서 JSON에 존재하지만 매핑되지 않은 속성을 무시하여, 데이터가 예상보다 많더라도 오류 없이 처리할 수 있도록 한다.

이러한 커스터마이징을 통해 Redis에 저장된 데이터를 보다 유연하고 안전하게 직렬화/역직렬화할 수 있으며, 다양한 데이터 타입을 Redis 캐시에 문제없이 저장하고 활용할 수 있다.

## 4. 로컬 캐시 설정 (Caffeine)
로컬 캐시 설정은 보조적으로 Caffeine을 사용하여 설정한다. `LocalCacheConfig` 클래스를 작성하고, `localCacheManager`에 `@Primary` 어노테이션을 사용하여 기본 캐시 매니저로 지정한다.

### 코드: 로컬 캐시 설정 (Caffeine)
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
    @Primary
    public CacheManager localCacheManager(List<CaffeineCache> caffeineCaches) {
        SimpleCacheManager cacheManager = new SimpleCacheManager();
        cacheManager.setCaches(caffeineCaches);
        return cacheManager;
    }
}
```

## 5. 서비스 메소드에서 캐시 사용하기
각 서비스 메소드에서 캐시 매니저를 명시적으로 지정하여 사용할 수 있다. 예를 들어, 글로벌 캐시와 로컬 캐시를 선택하여 사용할 수 있도록 설정한다.

- 글로벌 캐시 사용:

### 코드: 글로벌 캐시 사용
```java
@Cacheable(cacheNames = "allTranslations", key = "#locale", cacheManager = "globalCacheManager")
public List<Translation> getCachedTranslationsByLocale(Locale locale) {
    return translationRepository.findAllByLocale(locale);
}
```

- 로컬 캐시 사용:

### 코드: 로컬 캐시 사용
```java
@Transactional(readOnly = true)
@Cacheable(cacheNames = "locale", cacheManager = "localCacheManager")
public List<LocaleValue> getLocales() {
    return localeRepository.findAll();
}
```

위와 같이 각 서비스 메소드에서 `cacheManager` 속성을 사용하여 글로벌 캐시(`globalCacheManager`)와 로컬 캐시(`localCacheManager`)를 명시적으로 선택할 수 있다.

## 6. 마무리
스프링부트에서 Redis를 사용하여 글로벌 캐시를 설정하고, Caffeine을 보조적으로 로컬 캐시로 설정하는 방법을 살펴보았다. 두 캐시 매니저가 충돌하는 문제를 해결하기 위해 `localCacheManager`에 `@Primary` 어노테이션을 추가하고, 각 서비스 메소드에서 필요한 캐시 매니저를 명시적으로 지정하여 사용할 수 있도록 했다. 이를 통해 글로벌 캐시와 로컬 캐시를 효율적으로 활용하여 애플리케이션 성능을 최적화할 수 있다.

Redis와 Caffeine을 함께 사용하는 방식은 분산 환경과 로컬 환경 모두에서 성능을 극대화할 수 있는 좋은 방법이다. 이를 통해 반복적인 데이터베이스 호출을 줄이고, 애플리케이션의 응답 시간을 개선할 수 있다.