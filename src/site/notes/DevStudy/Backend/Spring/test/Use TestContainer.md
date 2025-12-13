---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Spring/test/Use TestContainer/","noteIcon":"","created":"2025-12-03T14:52:48.984+09:00","updated":"2025-12-13T10:18:24.331+09:00"}
---



> Spring Boot 테스트를 위해 실제 의존하는 DB를 dockerContainer로 띄어 테스틑하는 방법 



## 1. gradle 설정 
> Testcontainers를 사용하기 위해 `build.gradle` 파일에 필요한 의존성을 추가
```yaml
// Testcontainers 라이브러리 
testImplementation 'org.springframework.boot:spring-boot-testcontainers'  

// PostgreSQL 컨테이너를 위한 
testImplementation 'org.testcontainers:postgresql'

// Redis 사용 시 
// testImplementation 'org.testcontainers:redis'
```

## 2. 테스트 컨테이너 설정 


### 방법 1. `@TestConfiguration`을 이용한 Bean 등록 방식(권장)

>테스트 환경에서 사용될 `PostgreSQLContainer`를 Bean으로 미리 정의하여 중앙에서 관리하는 방법

```java
@TestConfiguration(proxyBeanMethods = false)  
class TestcontainersConfiguration {  
  
  @Bean  
  @ServiceConnection  //⭐ 3.1부터 도입된 핵심 애노테이션 
  PostgreSQLContainer<?> postgresContainer() {  
				// Docker 이미지 이름을 파싱하여 PostgreSQL 컨테이너 인스턴스를 생성
				return new PostgreSQLContainer<>(DockerImageName.parse("postgres:17.2-alpine")); 
				// 컨테이너 설정 추가 예시: 
				// .withDatabaseName("testdb") 
				// .withUsername("myuser") 
				// .withPassword("mypassword") 
				// .withInitScript("sql/init_schema.sql"); 
				// 초기 스키마 및 데이터 로드 스크립트 지정 
  }
```

#### 설명 
`@TestConfiguration(proxyBeanMethods = false)`
- **역할** : **테스트 전용 Config임을** Spring에게 알린다.
	- 이는 테스트 환경에서만 사용되는 Bean을 정의할 때 사용 
- `proxy~ = false`
	- @Bean으로 설정된 것이 프록시 객체를 생성하지 않도록 함
	- false 권장(in Test)
		- 테스트 환경에서는 빈간 순환참조, 복잡한 빈 라이프사이클 관리가 필요한 경우가 드물기 때문에 **`false`가  성능상 이점**이 있어 권장 

`@ServiceConnection` ⭐ 3.1부터 도입
- 역할 : `@Bean` 메서드가 반환하는 컨테이너는 Spring Boot의 **자동 구성(Auto-configuration) 메커니즘**에 의해 자동으로 데이터베이스 연결 정보를 인식하고 구성하도록 지시 
- 즉, `url-username-password` 등의 **DB 연결 속성을 수동으로 설정할 필요가 ❌**(Spring이 자동으로 주입하여 연결을 설정)
- **설정되는 `PostgresSQLContainer`의 기본 값들**
	- username = test
	- password = test
	- image = postgres
	- port = random()
- **사용자 정의도 가능 ❗**
	- 필요한 경우`withUsername()`, `withPassword()`, `withDatabaseName()` 등의 메소드를 통해 사용자 이름, 비밀번호, 데이터베이스 이름 등을 직접 지정할 수 있다


`new PostgreSQLContainer<>()`
- 새로운 PostgreSQL Docker 컨테이너 인스턴스를 생성


#### Config 활용 (`@Import`)
이전에 정의한 테스트용 설정 Bean인 `TestcontainersConfiguration`를 테스트 클래스에서 `@Import`하고 `@Autowired`를 사용하여 주입받아 활용할 수 있다.
```java 
@Import(TestContainerConfiguration.class)  
@SpringBootTest  
class ClosetApplicationTests {  
  
  Logger log = LoggerFactory.getLogger(ClosetApplicationTests.class);  
  
  @Autowired  
  PostgreSQLContainer<?> container;
 
	@Test  
  void contextLoads() {  
    container.start();  
    log.info("---{}---", container.getJdbcUrl());  
    log.info("---{}---", container.getUsername());  
    log.info("---{}---", container.getPassword());  
    container.stop();  
  }  
}
```

이전에 테스트용 설정 Bean인 `TestContainerConfiguration`를 **Import하고 `@Autowired`를 사용한다**
그 후, 각 메서드 or 클래스의 테스트가 시작될 때 start(), stop() 메서드를 사용하면 된다.

>[!tip] 직접 start, stop 할 필요가 없다
>- @ServiceConnection + @TestConfiguration 조합 시 Spring이 자동으로 처리해준다.
>- @ServiceConnection은 짱이다.



### 방법 2. 각 테스트 클래스 별 생성 
방법 1처럼, 따로 테스트용 Config를 하지 않고 각 클래스에서 PostgreSQLContainer를 생성하여 실행시킬 수 있다.
```java
@Container
@ServiceConnection
private static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>(
        DockerImageName.parse("postgres:17.2-alpine"));
```

`@Container` ⭐⭐⭐
- 역할 : 해당 필드(`postgres`)에 선언된 **Docker 컨테이너의 생명주기(Lifecycle)를 관리**하도록 지시하는 것
- 즉, **자동으로 컨테이너를 시작/중지하는 것** 
	- 테스트 클래스 시작 전 : 컨테이너 실행
	- 클래스의 모든 테스트 완료 후 : 컨테니어 중지 및 정리 
- **static인 이유❓** ⭐ 
	- `@Container`이 붙은 필드는 **`private static` 으로 선언하는 것이 권장**된다. 
	- **성능 최적화** ⭐⭐⭐
		- `static`으로 선언 시 해당 컨테이너 **인스턴스가** 테스트 클래스의 **모든 테스트 메서드에서 공유**된다.
		- 즉, 테스트 메서드 실행마다 새 컨테이너를 생성하고 중지시킬 필요가 없다는 것 




### 사용 예시

```java
@DataJpaTest  
@Testcontainers  
@Import({JpaAuditingConfiguration.class, TestContainerConfig.class})  
@ActiveProfiles("test")  
class UserRepositoryTest {


		@Autowired  
		private static PostgreSQLContainer<?> postgres;
```