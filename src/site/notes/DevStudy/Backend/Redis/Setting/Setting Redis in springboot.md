---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Redis/Setting/Setting Redis in springboot/","noteIcon":"","created":"2025-12-03T14:52:50.213+09:00","updated":"2025-12-13T10:25:14.544+09:00"}
---


## 1.  환경 세팅 
```java

dependencies {
  
  implementation 'org.springframework.boot:spring-boot-starter-data-redis'
}

```


```yaml




spring:
	...
	datasource:  
	  url: jdbc:mysql://${DB_HOST:host.docker.internal}:${DB_PORT:3307}/${DB_NAME:mydb}  
	  username: ${MYSQL_USERNAME:root}  
	  password: ${MYSQL_PASSWORD:pwd1234}  
	  driver-class-name: com.mysql.cj.jdbc.Driver

	data:
		redis:
			host: localhost
			prot: 6379

loggin:
	level:
		org.springframework.cache: trace
```

--- 
## 2.  Config 설정 

### 2.1.  기본 및 ConnectionFactory
```java
@Configuration
public class RedisConfig {
  @Value("${spring.data.redis.host}")
  private String host;

  @Value("${spring.data.redis.port}")
  private int port;

  @Bean
  public LettuceConnectionFactory redisConnectionFactory() {
    // Lettuce라는 라이브러리를 활용해 Redis 연결을 관리하는 객체를 생성하고
    // Redis 서버에 대한 정보(host, port)를 설정한다. 
    return new LettuceConnectionFactory(new RedisStandaloneConfiguration(host, port));
  }
}
```

--- 
### 2.2.  RedisTemplate
> 애노테이션없이 직접 save, delete 작업하는 경우 

```java
@Bean  
public RedisTemplate<String, String> redisTemplate() {  
  RedisTemplate<String, String> redisTemplate = new RedisTemplate<>();  
  
  redisTemplate.setConnectionFactory(redisConnectionFactory());  
  redisTemplate.setKeySerializer(new StringRedisSerializer());  
  redisTemplate.setValueSerializer(new StringRedisSerializer());  
  
  return redisTemplate;  
}
```

--- 

Object 타입 Value 사용 시 
```java 
@Bean  
public RedisTemplate<String, Object> redisTemplate() {  
  RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();  
  
  redisTemplate.setConnectionFactory(redisConnectionFactory());  
  redisTemplate.setKeySerializer(new StringRedisSerializer());  
  redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());  
  
  return redisTemplate;  
}

```

--- 
### 2.3.  트랜잭션 적용하고 싶으면 

`@Transaction` 사용 하고 싶으면 RedisTemplate에 
```java 
@Bean @Primary  
public RedisTemplate<String, String> redisTemplate() {  
  RedisTemplate<String, String> redisTemplate = new RedisTemplate<>();  
  ...
  redisTemplate.setEnableTransactionSupport(true);
```

--- 
### 2.4.  RedisCacheConfig 
> 애노테이션 사용해서 캐시하는 경우 
```java
@Configuration
@EnableCaching // Spring Boot의 캐싱 설정을 활성화
public class RedisCacheConfig {
  @Bean
  public CacheManager boardCacheManager(RedisConnectionFactory redisConnectionFactory) {
    RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration
        .defaultCacheConfig()
	      // Redis에 Key를 저장할 때 String으로 직렬화(변환)해서 저장
        .serializeKeysWith(
            RedisSerializationContext.SerializationPair.fromSerializer(
                new StringRedisSerializer()))
        // Redis에 Value를 저장할 때 Json으로 직렬화(변환)해서 저장
        .serializeValuesWith(
            RedisSerializationContext.SerializationPair.fromSerializer(
                new Jackson2JsonRedisSerializer<Object>(Object.class)
            )
        )
        // 데이터의 만료기간(TTL) 설정
        .entryTtl(Duration.ofMinutes(1L));

    return RedisCacheManager
        .RedisCacheManagerBuilder
        .fromConnectionFactory(redisConnectionFactory)
        .cacheDefaults(redisCacheConfiguration)
        .build();
  }
}
```
https://docs.spring.io/spring-boot/reference/io/caching.html#io.caching.provider.redis

--- 
## 3.  로직 
```java
@Cacheable(cacheNames = "getBoards", key = "'boards:page:' + #page + ':size:' + #size", cacheManager = "boardCacheManager") public List<Board> getBoards(int page, int size) {

		Pageable pageable = PageRequest.of(page - 1, size); 
		Page<Board> pageOfBoards = boardRepository.findAllByOrderByCreatedAtDesc(pageable); 
		return pageOfBoards.getContent(); 
}
```
- **속성 값** 
	- `cacheNames` : 캐시 이름을 설정
	- `key` : Redis에 저장할 Key의 이름을 설정
	- `cacheManager` : 사용할 `cacheManager`의 Bean 이름을 지정
	

>[!tip] @Cacheable 
>- Cache Aside 전략으로 캐싱이 적용
>- 즉, 요청이 들어오면 Redis 먼저 확인하는 것 
>- ![Pasted image 20250604221421.png](/img/user/supporter/image/Pasted%20image%2020250604221421.png)



---
## 4.  번외 : docker 설정

```dockerfile
FROM eclipse-temurin:17-jdk AS build  
  
WORKDIR /app  
  
COPY . .  
RUN chmod +x gradlew && ./gradlew clean build -x test;  
  
  
FROM openjdk:17-jdk-slim  
COPY --from=build /app/build/libs/*SNAPSHOT.jar app.jar  
EXPOSE 8080  
ENTRYPOINT ["java","-jar","/app.jar"]
```

```yaml
services:  
  server:  
    build: .  
    ports:  
      - 8080:8080  
    environment:  
      SPRING_PROFILES_ACTIVE: ${SPRING_PROFILES_ACTIVE:-dev}  
      SPRING_DATASOURCE_USERNAME: ${RDS_USERNAME:-root}  
      SPRING_DATASOURCE_PASSWORD: ${RDS_PASSWORD:-pwd1234}  
    depends_on:  
      cache-server:  
        condition: service_healthy  
  
  cache-server:  
    image: redis  
    ports:  
      - 6379:6379  
    healthcheck:  
      test: ["CMD", "redis-cli", "ping"]  
      interval: 5s  
      retries: 10  
  
#  my-db:  
#    image: mysql:latest  
#    environment:  
#      MYSQL_ROOT_PASSWORD: pwd1234  
#      MYSQL_DATABASE: mydb  
#    volumes:  
#      - my-db-data:/var/lib/mysql  
#  
#    ports:  
#      - 3307:3306  
#  
#volumes:  
#  my-db-data:
```