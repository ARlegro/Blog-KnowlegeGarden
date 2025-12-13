---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Spring/test/WebMvcTest/","noteIcon":"","created":"2025-12-03T14:52:48.965+09:00","updated":"2025-12-13T10:17:59.733+09:00"}
---


### 0.1.  초기 
```java
@Import(UserServiceImpl.class) //mocking 용  
@WebMvcTest(UserController.class)  
public class UserControllerTest {  
  
  @Autowired  
  MockMvcTester mockMvcTester; // 최신 버전  
  
  @Autowired  
  ObjectMapper objectMapper; // JSON 처리를 위한 필수  
  
  @MockitoBean  // 이게 최신버전 (deprecated : MockBean)  UserServiceImpl userService;  // 서비스 레이어 Mcok 처리
```


1. 테스트할 Controller 명시
```java 
@WebMvcTest(UserController.class)  
public class UserControllerTest {
```

2. Controller에서 의존하고 잇는 컴포넌트 Import
```java 
@Import(UserServiceImpl.class) //mocking 용  
@WebMvcTest(UserController.class)  
public class UserControllerTest {
```

3. 필요한 컴포넌트 Autowired 및 @MockitoBean 처리 

```java
@Autowired  
MockMvcTester mockMvcTester; // 최신 버전  
  
@Autowired  
ObjectMapper objectMapper; // JSON 처리를 위한 필수  
  
@MockitoBean  // 이게 최신버전 (deprecated : MockBean)UserServiceImpl userService;  // 서비스 레이어 Mcok 처리
```

Note : security test-config 방법 [[DevStudy/Backend/Spring/test/TestSecurityConfig\|TestSecurityConfig]]

### 0.2.  기본 예시 

아래는 검증 없이 콘솔만 보기 위해 print()만 했다/
```java 
@Import(UserServiceImpl.class)
@WebMvcTest(UserController.class)  
public class UserControllerTest {  
  
  @Autowired MockMvcTester mockMvcTester;
  @Autowired ObjectMapper objectMapper;   
  @MockitoBean UserServiceImpl userService;  
  
  @Test  
  void createUser() throws JsonProcessingException {  
    var newUser = new UserCreateRequest(  
        "new username",  
        "new email",  
        "new password"  
    );  
  
    mockMvcTester.post()  
        .uri("/api/users")  
        .contentType(String.valueOf(MediaType.APPLICATION_JSON))  
        .content(objectMapper.writeValueAsString(newUser))  
        .assertThat().apply(print());  
  }
```

> 결과

```java
MockHttpServletRequest:
      HTTP Method = POST
      Request URI = /api/users
       Parameters = {}
          Headers = [Content-Type:"application/json;charset=UTF-8", Content-Length:"69"]
             Body = {"name":"new username","email":"new email","password":"new password"}
    Session Attrs = {org.springframework.security.web.csrf.HttpSessionCsrfTokenRepository.CSRF_TOKEN=org.springframework.security.web.csrf.DefaultCsrfToken@3eadad14}

Handler:
             Type = null

Async:
    Async started = false
     Async result = null

Resolved Exception:
             Type = null

ModelAndView:
        View name = null
             View = null
            Model = null

FlashMap:
       Attributes = null

MockHttpServletResponse:
           Status = 403
    Error message = Forbidden
          Headers = [X-Content-Type-Options:"nosniff", X-XSS-Protection:"0", Cache-Control:"no-cache, no-store, max-age=0, must-revalidate", Pragma:"no-cache", Expires:"0", X-Frame-Options:"DENY"]
     Content type = null
             Body = 
    Forwarded URL = null
   Redirected URL = null
          Cookies = []
> Task :test
```

Request부분을 보면 내가 입력한대로 나왔다. 


### 0.3.  단일 검증

```java
mockMvcTester.post()  
    .uri("/api/users")  
    //.with(csrf()) // CSRF 토큰 추가 (TestConfig에 놓았으면 필요 없음)  
    .contentType(MediaType.APPLICATION_JSON.toString())  
    .content(objectMapper.writeValueAsString(newUser))  
    .assertThat().apply(print())  
    .hasStatusOk()  
    .bodyJson().extractingPath("$.id") // json으로 변환해준다? jsonPath 라이브러리  
    .isEqualTo(id.toString());
```
- JsonPath 라이브러리를 이용해서 body를 json으로 변환시키고 $와 asseertJ의 api를 활용해서 비교 


>[!EXAMPLE] extractingPath()
>- JSON 응답에서 특정 경로의 값을 추출하는 메서드
>- 값을 추출하고 타입을 변환 시키려면 아래처럼 하면 된다.
>- 표현식 정리
>	- `$` : 루트 객체
>	- `.` : 자식 노드
>	- `[]` : 배열 인덱스
>	- `[*]` : 모든 배열 요소
```java
// 문자열로 변환
.extractingPath("$.name").asString().isEqualTo("John")

// 숫자로 변환 
.extractingPath("$.age").asNumber().isEqualTo(25) 

// Boolean으로 변환
.extractingPath("$.isActive").asBoolean().isTrue()
```




### 0.4.  다중 검증 

#### 0.4.1.  방법 1. 각 필드 개별 검증 
```java 
var result = mockMvcTester.post()  
    .uri("/api/users")  
    .contentType(MediaType.APPLICATION_JSON.toString())  
    .content(objectMapper.writeValueAsString(newUser))  
    .assertThat().apply(print())  
    .hasStatusOk().bodyJson();  
  
result.extractingPath("$.id").isEqualTo(id.toString());  
result.extractingPath("$.email").isEqualTo(email);  
result.extractingPath("$.name").isEqualTo(name);  
result.extractingPath("$.role").isEqualTo(UserRole.USER.toString());  
result.extractingPath("$.locked").isEqualTo(false);
```

#### 0.4.2.  방법 2. Converto()로 객체 변환 후 검증 

```java
var result = mockMvcTester.post()  
    .uri("/api/users")  
    //.with(csrf()) // CSRF 토큰 추가 (TestConfig에 놓았으면 필요 없음)  
    .contentType(MediaType.APPLICATION_JSON.toString())  
    .content(objectMapper.writeValueAsString(newUser))  
    .assertThat().apply(print())  
    .hasStatusOk().bodyJson()  
    .convertTo(UserDto.class)  
        .satisfies(userDto -> {  
          assertThat(userDto.id()).isEqualTo(id);  
          assertThat(userDto.createdAt()).isEqualTo(now);  
          assertThat(userDto.email()).isEqualTo(email);  
          assertThat(userDto.name()).isEqualTo(name);  
          assertThat(userDto.role()).isEqualTo(UserRole.USER);  
          assertThat(userDto.locked()).isEqualTo(false);  
        });
```
