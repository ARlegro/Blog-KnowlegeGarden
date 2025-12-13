---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Spring/Java MailSender in Spring Doc/","noteIcon":"","created":"2025-12-03T14:52:47.707+09:00","updated":"2025-12-13T09:26:28.913+09:00"}
---


https://docs.spring.io/spring-framework/reference/integration/email.html

>[!tip] 참고 : [[DevStudy/Backend/Spring/EmailDev\|EmailDev]] ➡ 연습하는 곳 

---
### JavaMailsender 인터페이스 
> 이메일 전송 기능을 추상화하여 제공하는 핵심 인터페이스


---
#### 순수 JavaMail API의 단점 💢
- 자바에서 제공하는 JavaMail API는 매우 강력한 기능이다.
- But, 직접 사용하려면 여러 번거로움이 필요하다
	- Session 객체 관리
	- MimeMessage 생성
	- Transport 사용 등 

> 추상화의 귀재 스프링은 이러한 복잡한 과정을 추상화하여 개발자가 이메일 내용 작성 및 전송에만 집중할 수 있도록 한다.


---
#### 추상화된 JavaMailSender의 장점 ✅
1. **Spring 환경과 통합**
	- DI도 가능하고
	- AOP도 가능 
	- @Bean을 통해 커스텀 설정도 가능 
2. **테스트 용이** 
	- 인터페이스 기반이라 메일 서버에 의존하지 않고 Mock객체를 사용하여 이메일 전송 로직 단위테스트가 용이하다 
3. **다양한 구현체 지원**
	- 다양한 프로토콜(SMTP, SMTPS 등)을 지원하는 구현체가 다양해서 쉽게 교체 가능 


---
#### 구현체 
> 대표적이고 일반적인 구현체 = `JavaMailSenderImpl`


---

### 주요 메서드 

*mime(Multipurpose Internet Mail Extensions* : 이메일에서 text 외의 다양한 형식 (오디오, 비디오, 이미지 등)을 지원하기 위해 사용되는 확장 기술 

---
#### 1. MimeMessage createMimeMessage();
> 새로운 Bean MimeMessage 객체를 생성
- 이메일의 header, 수신자, 제목, 본문 등을 직접 설정 가능 
- 보통 `MimeMessageHelper`와 함께 사용된다.
- 활용 : 주로, 복잡한 커스터마이징 메일 보낼 시 사용 ex. HTML 콘텐츠, 첨부파일, 이미지 등 

```java 
@Service  
@RequiredArgsConstructor  
public class EmailService {  
  
  private final JavaMailSender javaMailSender;  
  
  public void sendEmail(~~~) throws MessagingException {  
  
    MimeMessage mimeMessage = javaMailSender.createMimeMessage();  
    MimeMessageHelper helper = new MimeMessageHelper(mimeMessage,  
        MimeMessageHelper.MULTIPART_MODE_MIXED, UTF_8.name());  
        
		// 
    // elper.setFrom("example123@naver.com"); 설정에서 했으므로 생략 가능 
    helper.setTo("icb1696@naver.com");  
    helper.setSubject("TEST 제목 입니다.");  
    helper.setText("TEST 내용 입니다.", true);  
      
    // 첨부파일 시 : helper에서 true or MULTIPART_MODE 설정 必
		// FileSystemResource file = new FileSystemResource(new File("path/to/my/file.pdf"));  
    // helper.addAttachment("document.pdf", file);

			
    javaMailSender.send(mimeMessage);  
  }
```


>[!tip] MultiPart or HTML 설정 시  (in spring doc example)
```JAVA
MimeMessage message = sender.createMimeMessage();

// 멀티파트 메시지가 필요함을 나타내기 위해 true 플래그를 사용
MimeMessageHelper helper = new MimeMessageHelper(message, true);

helper.setTo("test@host.com");
// 포함된 텍스트가 HTML임을 나타내기 위해 true 플래그를 사용합니다.
helper.setText("<html><body><img src='cid:identifier1234'></body></html>", true);

// 유명한 Windows 샘플 파일을 포함합니다 (이번에는 c:/에 복사됨).
FileSystemResource res = new FileSystemResource(new File("c:/Sample.jpg"));
helper.addInline("identifier1234", res);
// addAttachment = 첨부파일
// addInLine = 본문에 삽입 

sender.send(message);
```
> [!WARNING] 인라인 리소스는 무조건 텍스트 뒤에 세팅 必 




---
#### 2. MimeMessage createMimeMessage(InputStream ~);
> 주어진 InputStream으로부터 MimeMessage 객체를 새성
> - 주로 미리 정의된 이미지 구조(템플릿)를 로드하여 사용할 때 유용 

---
#### 3. void send(MimeMessage ~)
> 주어진 MimeMessage 객체를 전송 
- 실제로 메일 서버를 통해 이메일 발송 

---
#### 4. void send(MimeMessage... )
> 여러 개의 MimeMessage 객체를 일괄 전송 
- 여러 사용자에게 다른 내용의 이메일을 보낼 때 성능상 이점 
- 대량 메일 발송 시 각 메시지를 개별적으로 보내기 

---
#### 5. void send(SimpleMailMessage ~ )
> SimpleMailMessage 객체를 전송 
- text 기반의 간단한 이메일을 보낼 때 사용되는 Spring 유틸 클래스 
- ❌첨부 파일, HTML 컨텐츠를 지원하지 않는다.
- 활용 : 인증 코드, 알림 메시지 등  (단순 텍스트 이메일)

```java
SimpleMailMessage simpleMailMessage = new SimpleMailMessage();  
simpleMailMessage.setTo(user.getEmail());  
simpleMailMessage.setSubject("[임시 비밀번호 발급]");  
String message = "임시 비밀번호 발급 메일입니다. \n [임시 비밀번호] : " + tempPassword;  
simpleMailMessage.setText(message);  
  
try {  
  javaMailSender.send(simpleMailMessage);
```

---
#### 6. void send(SimpleMailMessage... )
> 여러 개의 SimpleMailMessage 객체를 일괄 전송 
- 여러 사용자에게 보낼 때 유용


---
### 주요 환경설정

```properties
spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=your_email@gmail.com
spring.mail.password=your_app_password
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
```
- 이런 설정 하면 JavaMailSender 빈이 구성된다.



### 단위 테스트 방법 


```java
@ExtendWith(MockitoExtension.class)  
public class MailSendProcessorTest {  
  
  @Mock  
  private JavaMailSender javaMailSender;  
  
  @InjectMocks  
  private MailSendProcessor mailSendProcessor;  
  
  @Test  
  public void sendMail() {  
    String to = "toto123@naver.com";  
    String tempPassword = "tempPassword";  
    mailSendProcessor.sendTemporaryPassword(to, tempPassword);  
  
    verify(javaMailSender, times(1)).send(any(SimpleMailMessage.class));  
}
```


---
### 비동기 
> Email 전송같이 오래 걸리가너 외부 I/O가 필요한 작업은 비동기적으로 실행되면 좋다.

#Async 

---
#### @Async 필요성 
- Email 전송 작업은 네트워크 지연 등으로 인해 수십초까지 걸릴 수도 있다. 이는 사용자 경험을 매우 저하시키고 불필요한 스레드 점유를 발생시킨다.
- **@Async를 사용하여 별도 실행**
	- 이메일 전송과 같은 작업은 별도의 Thread에서 실행하고, 원래 호출 Thread는 즉시 작업을 처리
	- 이로 인해, 불필요하게 **Thread를 점유하는 문제를 방지** 

>[!tip] @Async 작동 방식
>- @Async가 붙은 메서드는 새로운 Thread or 미리 구성된 Thread pool에서 실행되도록 스케줄링 됨
>- 기존 호출 Thread는 즉시 반환되어 다음 작업 수행 

---
#### 설정 
> 사용하려면 몇 가지 설정 必

**메인 클래스 or 별도의 Config 클래스에 `@EnableAsync` 추가** 
```java
@SpringBootApplication
@EnableAsync // 비동기 기능 활성화
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
```

**Executor 설정** 
- 💢**미설정 시** : SimpleAsyncTaskExecutor 사용 
	- But, 이 executor는 Thread를 무제한으로 생성할 수 있어 위험하다.
- ✅**권장 설정** : `ThreadPoolTaskExecutor`와 같이 스레드 풀을 관리하는 Executor를 명시
	```java
	@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);  // 기본 스레드 수
        executor.setMaxPoolSize(10);  // 최대 스레드 수
        executor.setQueueCapacity(25); // 큐 용량 
        executor.setThreadNamePrefix("MyAsyncThread-"); // 스레드 이름 접두사
        executor.initialize();
        return executor;
    }

    // 비동기 작업에서 발생하는 예외를 처리하기 위한 예외 핸들러 설정 (Optional)
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new SimpleAsyncUncaughtExceptionHandler();
    }
	```
---
#### 사용 

```java 
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

@Service
public class EmailService {

    @Autowired
    private JavaMailSender mailSender;

    // 이메일 전송 메서드를 비동기적으로 실행
    @Async <<<<
    public void sendSimpleEmail(String to, String subject, String text) {
        
				SimpleMailMessage message = new SimpleMailMessage();
				message.setTo(to);
				message.setSubject(subject);
				message.setText(text);
```
---
#### @Async 사용 시 알아둬야 할 점 

1. **pulbic이어야 한다.**
	- 이유 : AOP가 적용되야 하므로 
2. **동일 클래스 내에서 호출 시 비동기 적용❌**
	- 이유 : AOP가 해당 호출을 가로채지 못하기 때문 
3. **반환 타입** : `void` or `Futer<T>`
4. **비동기에서 발생한 예외 호출 메서드로 전파되지 않는다.**
	- 따라서, `try-catch` or `AsyncUncaughtExceptionHandler`를 구현하여 Globla 비동기 예외처리 必
5. **트랜잭션 미적용**
	- 비동기로 실행되는 Thread는 기존 호출 스레드의 `Transaction Context`를 상속받지 않는다.
	- **헷갈릴 점**
		- ✅@Async + @Transaction은 비동기 스레드도 Transaction 적용됨 
		- ❌ @Async + 다른 외부 호출(@Transaction 중인)
  > 따라서, **DB작업이 필요한 `@Async`메서드의 경우 별도로 `@Transaction` 선언 필수**

