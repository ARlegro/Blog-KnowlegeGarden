---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Spring/Security/CORS & CSRF/CORs/","noteIcon":"","created":"2025-05-27T23:07:18.780+09:00","updated":"2025-12-24T21:27:27.487+09:00"}
---



> CORS (Cross-Origin Resource Sharing)

#JsonIgnore

>[!TIP] How to filter specified field whe response ➡ @JsonIgnore
```java
@JsonIgnore  
@Column(name = "create_dt")  
private Date createDt;  
  
@JsonIgnore  
@Column(name = "update_dt")  
private Date updateDt;
```

## 1.  Before start

#CORS 
Sometime, I may **encounter this error** when frontend **tries to access an API** hosted on a **different port** 
```shell
Access to XMLHttpRequest at 'http://localhost:8080/notices' from origin 
'http://localhost:4200' has been blocked by CORS policy:
Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource.Understand this error
:8080/notices:1 
            
           Failed to load resource: net::ERR_FAILEDUnderstand this error
core.mjs:10614 ERROR HttpErrorResponse
```
![Pasted image 20250523150542.png](/img/user/supporter/image/Pasted%20image%2020250523150542.png)
> The communication between 2 applications(4200, 8080) has been blocked by CORS policy
> `has benn blocked by CORS policy` 
- This happens **only in browsers(UI application)**, not tools like Postman 
- **Browsers enforce CORS** as a client-side security policy to **prevent** unauthorized data access


--- 
## 2.  What is CORS 
 > Cross-Origin Resource Sharing
 > CORS = protocol

#Browser-policy  #prevent-website

---
### 2.1.  Concept
CORS is a **protocol** that **allows** JS running in a **browser to make requests to a different origin** 
(So, Since **Postman is not a browser**, no need to worry about CORS)
It's a defensive mechanism agains malicious cross-site interactions

By default, **browsers block cross-origin HTTP requests** for security reasons.
 - 따라서 이런 경우 따로 설정이 없다면 다른 자원을 가져올 수 없어진다.

>[!QUESTION] Note : What is origin
>- URL or Domain name
>- Origin is defined by a combination of 3 parameters
>	1. protocol (ex. HTTP, HTTPS)
>	2. domain (ex. localhost)
>	3. port (ex. 8080)
>	> ex. `https://www.google.com` 
>- ⭐So, If  anyone of these three is different, **it is considered a different origin** 
>> 2 dfifferent origin ➡ 2 different applications


>[!tip] Conclusion
>- ❌ CORS != security issue/attack, protecting server
>- ✅ **CORS == protection to stop sharing data/info between different origin**


--- 
### 2.2.  Why user CORS 
- It is trying to **be cautions to award some security threats** from the hackers with different origin
- **Browsers prevent websites** from feely sending ro requestting data to/from other websites

![Pasted image 20250523153751.png](/img/user/supporter/image/Pasted%20image%2020250523153751.png) 


--- 
## 3.  How to configure CORS to allow specific origin 

### 3.1.  ❌Bad Practice. @CrossOrigin( ~ ~) 
```java 
1. 구체적 경로 
@CrossOrigin(origin = "http://localhost:4200") 

2. 전체 허용 
@CrossOrigin(origin = "*")


@CrossOrigin(origins = "http://localhost:4200")
@RestController
public class NoticeController {
    ...
}
```
 Annotaions is used above the controllser as a class-level 

**💢Drawback**
- You must annotate each controller manually
- **it can be a tedious job**  ➡ Not good for large projects

> So this method is not used 

### 3.2.  ✅Best Practice. Configure in Spring Security ⭐⭐⭐

By http.cors( ~), i can cofigure CORS settings 
```java 
@Bean  
SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {  
    http.cors(corsConfig -> corsConfig.configurationSource(new CorsConfigurationSource() {  
            @Override  
            public CorsConfiguration getCorsConfiguration(HttpServletRequest request) {  
                CorsConfiguration config = new CorsConfiguration();  
                config.setAllowedOrigins(Collections.singletonList("http://localhost:4200"));  
                config.setAllowedMethods(Collections.singletonList("*"));  
                config.setAllowedHeaders(Collections.singletonList("*"));  
                config.setAllowCredentials(true); // trying to enable accepting the user credentials and cookies  
                config.setMaxAge(3600L); // 1 hour  
                return config;  
            }  
        }))  
        .sessionManagement( smc ->
```
> Use .configurationSource() method ➡ Pass CorsConfiguraionSource + getCorsConfiguration method

✅**Flow** 
1. configure CorsConfigurationSource
2. Implement corsConfigurationSorce's method
	- Construct CorsConfiguration
	- Set origin, methods, credentials, headers  etc 

>Note : singletonList is used when use a single value 

✅**Explanation of method** 
1. **setAllowedOrigins**
	- Specifies which origins are allowed to access resources 
	  
2. **setAllowedHeaders**
	- Headers the client is allowed to send
	- if i don't mention allowedHeaders, i need to provide the specific header name
	  
3. **setAllowCredentials(true)**
	- Enables supprots for cookies/sessions/tokens 
		- Spring Security가 이 값을 **true**로 넣으면 모든 CORS 응답에 `Access-Control-Allow-Credentials: true` 헤더가 추가
		- 브라우저는 이 헤더를 받으면 **쿠키·세션 ID·Authorization 헤더 같은 “자격 증명(credentials)”을 포함해도 된다**고 판단
			- JWT를 헤더 or JSON에 직접 담아서 보내는 방식이라면 이 설정 필요 없음 -> 단지 Cookie, Session 방식용
	- When set to `true`, the client must include credentials 
	- When set to `false`
		```js
		// 프론트 
		fetch("http://localhost:8080/api/user", { credentials: "include" // 쿠키 보내고 싶어! }); 
		``` 
		``` 브라우저: ❌ 안돼. 서버가 credentials 허용 안 함 → 쿠키 안 보냄
		```
	  
4. **setMaxAge(3600L)**
	- Setting How long a CORS **pre-flight response can be cached** by the browser
	- ex. 3600L ➡ In hours, browser don't need to send pre-flight Cuz cashing 
	- this **optimizes** performance by **letting browser cache the CORS permission response** 


>[!QUESTION] ❓어떻게 브라우저가 이 설정을 인식할까?
>- 브라우저는 실제 Front의 API request를 보내기 전에 **pre-flight request**를 보낸다.
>- 이때, 브라우저는 CORS related configuraiton 을 확인한다.
>- 만약 Backend에서 해당 API로 CORS protect하도록 설정되어있었다면, 브라우저가 Block하고 CORS 에러를 낸다.
> > preflight request is fail ➡ actual request fail

---
## 4.  Test Result 
### 4.1.  Success image

![Pasted image 20250523170500.png](/img/user/supporter/image/Pasted%20image%2020250523170500.png)

![Pasted image 20250523170619.png](/img/user/supporter/image/Pasted%20image%2020250523170619.png)

I am not getting CORS ERROR 

---
### 4.2.  See Response header 

![Pasted image 20250523171111.png](/img/user/supporter/image/Pasted%20image%2020250523171111.png)
>[!tip] Response's header 는 어디서 정해진걸까??
>- CorsFilter라는 클래스에 있다.

```java
public class CorsFilter extends OncePerRequestFilter {  
  private final CorsConfigurationSource configSource;  
  private CorsProcessor processor = new DefaultCorsProcessor();  
  
	...
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {  
    CorsConfiguration corsConfiguration = this.configSource.getCorsConfiguration(request);  
    boolean isValid = this.processor.processRequest(corsConfiguration, request, response);  
    if (isValid && !CorsUtils.isPreFlightRequest(request)) {  
      filterChain.doFilter(request, response);  
    }  
  }
```
- This filter is responsible in loading all the CORS relate config into the response header 


---
### 4.3.  실패 케이스 
localhost 가 아닌 실제 ip로 접속하면??
> `http://127.0.0.1:4200/notices`

![Pasted image 20250523170855.png](/img/user/supporter/image/Pasted%20image%2020250523170855.png)
