---
title: "Spring Boot Redis Cache"    
layout: single    
read_time: true    
comments: true   
categories: 
 - project  
toc: true    
toc_sticky: true    
toc_label: contents    
description: Spring Boot Redis Cache 적용
last_modified_at: 2021-12-16     
---

## Cache란?

캐시란 자주 사용되는 데이터를 메모리등의 공간에 담아두고 같은 데이터를 요청 했을 때 해당 공간에서 바로 꺼내 제공해주는 기술을 말합니다.

![1](/assets/image/springboot_redis_cache/1.png)

이를 시각화하면 위 이미지와 같습니다.

애플리케이션은 바로 DB에 접근해서 데이터를 가져오는 것이 아니라 캐시메모리에 먼저 접근한 후 원하는 데이터가 있는지 확인하고 있으면 캐시에서 없으면 DB에서 데이터를 가져오도록 합니다.

이러한 구조를 적용하게되면 자주 사용되는 데이터를 가져올 때 데이터베이스에서 조회하는 시간을 절약할 수 있고 데이터베이스에 가해지는 부담 또한 줄여줄 수 있습니다.



## Redis Cache 적용

Redis는 Key-Value구조의 In-Memory 데이터베이스 입니다.

분산 환경에서의 캐시 서비스를 구현하기위해 Redis를 통해 스프링 애플리케이션에 캐시 기능을 적용해보겠습니다.



**pom.xml에 spring-boot-starter-data-redis추가**

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```



**application.properties에 redis 연결 정보 추가**

```properties
spring.redis.cache.host=IP
spring.redis.cache.port=PORT
```



**Main 메소드가 존재하는 클래스에 @EnableCaching 어노테이션 추가**

```java
@SpringBootApplication
@EnableCaching
public class CachingApplication {

    public static void main(String[] args) {
        SpringApplication.run(CachingApplication.class, args);
    }
}
```



**Redis 연결을 위한 빈 설정**

```java
@Configuration
public class CacheConfig {
    @Value("${spring.redis.cache.host}")
    private String cacheHostName;

    @Value("${spring.redis.cache.port}")
    private int cachePort;

    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.registerModules(new JavaTimeModule(), new Jdk8Module());
        objectMapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);

        return objectMapper;
    }

    @Bean
    public ObjectMapper cacheObjectMapper() {
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.activateDefaultTyping(
            BasicPolymorphicTypeValidator.builder().allowIfSubType(Object.class).build(),
            ObjectMapper.DefaultTyping.NON_FINAL);
        objectMapper.registerModules(new JavaTimeModule(), new Jdk8Module());
        objectMapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);

        return objectMapper;
    }

    @Bean
    public RedisConnectionFactory redisCacheConnectionFactory() {
        RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();
        redisStandaloneConfiguration.setHostName(cacheHostName);
        redisStandaloneConfiguration.setPort(cachePort);
        redisStandaloneConfiguration.setPassword(cachePassword);

        return new LettuceConnectionFactory(redisStandaloneConfiguration);
    }

    @Bean
    public CacheManager redisCacheManager(RedisConnectionFactory redisCacheConnectionFactory, ObjectMapper cacheObjectMapper) {
        RedisCacheConfiguration configuration = RedisCacheConfiguration.defaultCacheConfig()
            .disableCachingNullValues()
            .entryTtl(Duration.ofSeconds(60))
            .serializeKeysWith(RedisSerializationContext
                .SerializationPair
                .fromSerializer(new StringRedisSerializer())).serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer(cacheObjectMapper)));

        return RedisCacheManager.RedisCacheManagerBuilder.fromConnectionFactory(redisCacheConnectionFactory)
            .cacheDefaults(configuration).build();
    }
}
```

ObjectMapper 두 개를 빈으로 등록한 이유는 뒤에 설명하도록 하겠습니다.

이제 캐시를 사용하기위한 설정이 완료되었습니다 다음 처럼 게시글을 조회하는 서비스 메소드에 @Cacheable 어노테이션을 붙여주면 최초에 한번만 데이터베이스에서 조회하고 그 뒤에는 Redis에서 조회하는 모습을 볼 수 있습니다.

```java
@Service
@RequiredArgsConstructor
public class PostService {

    private final PostRepository postRepository;

    @Cacheable(value = "post", key = "#postId")
    public PostResponseDto getPost(long postId) {
        Optional<Post> post = postRepository.findById(postId);

        return PostResponseDto.of(post.orElseThrow(NullPointerException::new));
    }
}
```

![2](/assets/image/springboot_redis_cache/2.png)

위 이미지는 Redis에 데이터가 저장된 모습입니다.



이제 ObjectMapper 빈을 두 개를 정의한 이유에 대해 설명하겠습니다.

만약 아무런 설정정보를 추가하지 않은 ObjectMapper를 사용해 Java Object를 Redis와 통신하며 직렬화/역직렬화 하게되면 어떻게 될까요?

아래와 같은 Object만을 사용하면 크게 문제가 발생하지 않을 것 입니다.

```java
public class Post {
    private Long postId;
    private String title;
    private String contents;
}
```

근데 만약 아래 처럼 Object안에 List와 같은 Object가 추가되면 어떻게될까요?

```java
public class Post {
    private Long postId;
    private String title;
    private String contents;

    List<Comment> comments;
}
```

아마 Redis에 저장까지는 되겠지만 다시 역직렬화해 Java Object로 만들려할 때 ClassCastException이 발생할 것 입니다.

아래는 실제 발생한 예외 입니다.

```java
java.lang.ClassCastException: class java.util.ArrayList cannot be cast to class com.example.jpastudy.dto.PostResponseDto (java.util.ArrayList is in module java.base of loader 'bootstrap'; com.example.jpastudy.dto.PostResponseDto is in unnamed module of loader 'app')
```

ObjectMapper의 activateDefaultTyping() 메소드를 사용해 설정을 해주면 위 문제를 해결할 수 있습니다. 해당 메소드는 ObjectMapper를 사용해 객체를 Json타입으로 변경할 때 유형정보를 추가해주는 역할을 합니다.

실제로 아래의 저장된 데이터를 보면 com.example.* 처럼 유형정보가 포함되어 저장된 모습을 확인할 수 있습니다.

```json
[
	"com.example.jpastudy.dto.PostResponseDto",
	{
		"postId": 7,
		"title": "TITLE",
		"contents": "CONTENTS",
		"comments": [
			"java.util.ArrayList",
			[
				[
					"com.example.jpastudy.dto.CommentResponseDto",
					{
						"commentId": 1,
						"contents": "contents"
					}
				]
			]
		]
	}
]
```

이러한 유형정보가 포함되어있기 때문에 다시 역직렬화 할 때 ClassCastException이 발생하지 않게 됩니다.

그러면 유형정보를 포함하는 설정을 추가한 ObjectMapper만 사용하면 되지 않을까라는 생각을 할 수 있지만 ObjectMapper가 Redis와 데이터를 주고받을때만 사용되는 것이 아니기 때문에 하나의 ObjectMapper가 더 필요한 것입니다.



만약 유형정보를 포함하는 ObjectMapper만 사용할 경우 응답데이터에 까지 유형정보가 포함될 수 있습니다.

객체를 Json으로 변환해서 응답데이터의 본문에 추가해 응답을 보낼 때 객체를 Json으로 변환하는 과정은 MappingJackson2HttpMessageConverter라는 메시지 컨버터가 수행하는데 해당 컨버터가 ObjectMapper를 사용하기 때문에 그러한 결과가 나오게 되는것입니다.

그래서 캐시용 Redis와 데이터를 주고받을때는 유형정보를 포함하는 ObjectMapper를 메시지컨버터는 일반 ObjectMapper를 사용하도록 두 개의 ObjectMapper를 정의한 것 입니다.
