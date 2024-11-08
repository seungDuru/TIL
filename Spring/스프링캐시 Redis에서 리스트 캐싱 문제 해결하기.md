# 스프링캐시 Redis에서 리스트 캐싱 문제 해결하기

스프링부트에서 Redis를 글로벌 캐시로 사용하고, 리스트 데이터를 캐싱하는 과정에서 발생하는 문제와 그 해결 방법에 대해 알아보자. Redis를 캐시로 사용할 때 직렬화/역직렬화를 처리하기 위해 주로 GenericJackson2JsonRedisSerializer를 사용한다. 이 방식은 간편하게 데이터를 직렬화하고 저장할 수 있지만, 리스트 형태의 데이터를 저장할 때 문제가 발생할 수 있다. 이 글에서는 그 문제의 원인과 해결 방안을 함께 살펴본다.

## 1. GenericJackson2JsonRedisSerializer와 리스트 캐싱 문제

GenericJackson2JsonRedisSerializer는 데이터를 직렬화할 때 클래스 정보나 기타 부가적인 정보를 함께 저장한다. 이러한 방식은 일반적인 단일 객체를 캐싱할 때는 유용하지만, 리스트 형태의 데이터를 캐싱할 경우 문제가 생길 수 있다. 리스트로 저장된 데이터가 역직렬화될 때, 이 추가적인 정보들로 인해 Jackson이 적절한 타입으로 매핑하지 못하고 오류를 발생시키는 경우가 있다.

이는 주로 Redis에 저장된 데이터에 대한 타입 정보를 잭슨이 해석하는 과정에서 발생하는데, 클래스 타입 정보를 잃어버리거나, 혹은 리스트의 원소 타입을 제대로 인식하지 못하는 문제가 있다. 따라서 리스트 데이터를 캐싱할 때는 GenericJackson2JsonRedisSerializer를 사용하면 기대한 대로 역직렬화가 되지 않을 수 있다.

## 2. 해결 방법: 리스트 데이터를 Wrapping 하기 

이 문제를 해결하기 위해 리스트 데이터를 별도의 객체로 한번 더 감싸는 방법을 사용할 수 있다. 리스트를 바로 Redis에 저장하는 대신, 리스트를 포함하는 Wrapper 클래스를 정의하여 저장하는 것이다. 이렇게 하면 Redis에 저장될 때 리스트의 타입 정보를 잃지 않고, 역직렬화할 때도 Jackson이 올바르게 타입을 인식할 수 있다.

아래 코드는 리스트 데이터를 감싸기 위한 `RedisListWrapper` 클래스와 이를 활용한 캐시 저장 방식을 보여준다.

```java
@Cacheable(cacheNames = "allTranslations", key = "#locale", cacheManager = "globalCacheManager")
public RedisListWrapper<Translation> getCachedTranslationsByLocale(Locale locale) {
    return translationRepository.findAllByLocale(locale);
}
```

```java
@Getter
@NoArgsConstructor
public class RedisListWrapper<T> implements Serializable {

    private List<T> list;

    @Builder
    public RedisListWrapper(List<T> list) {
        this.list = list;
    }

    public static <T> RedisListWrapper<T> of(List<T> list) {
        return RedisListWrapper.<T>builder()
            .list(list)
            .build();
    }
}
```

위 코드에서 `getCachedTranslationsByLocale` 메소드는 캐시 저장 시 리스트 데이터를 `RedisListWrapper`로 감싸서 저장한다. `RedisListWrapper` 클래스는 리스트를 포함하는 간단한 Wrapper 객체로, 직렬화/역직렬화 시 리스트의 타입 정보를 유지하도록 돕는다. 이 방식으로 저장된 데이터는 Jackson이 문제없이 역직렬화할 수 있으며, 캐시에서 데이터를 가져올 때도 안전하게 리스트 형태로 변환된다.

사용처에서는 캐시에서 꺼낸 데이터를 `getList()` 메소드를 통해 실제 리스트 데이터로 접근할 수 있다. 이렇게 Wrapping 방식을 사용하면 GenericJackson2JsonRedisSerializer를 사용할 때 발생하는 리스트 타입의 역직렬화 문제를 손쉽게 해결할 수 있다.

## 3. 마무리

Redis를 글로벌 캐시로 사용하면서 GenericJackson2JsonRedisSerializer를 사용할 때 발생할 수 있는 리스트 형태의 데이터 타입 인식 문제를 해결하기 위해 간단한 Wrapper 객체를 사용하는 것은 데이터 Wrapping 방식이 매우 유용하다. 특히 간단히 사용할 수 있어서 매우 편하다.

#spring