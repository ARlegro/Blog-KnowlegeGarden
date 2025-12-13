---
{"dg-publish":true,"permalink":"/DevStudy/Backend/RM/Intro/Start with Spring/","noteIcon":"","created":"2025-12-03T14:52:50.000+09:00","updated":"2025-12-13T10:22:07.193+09:00"}
---



Spring은 AMQP프로토콜을 제공하며 RabbitMQ와 ??

```yaml
spring:  
  rabbitmq:  
    host: localhost  
    port: 5672  
    username: guest  
    password: guest
```
이건 기본값이다.


RabbitMQ플러그인 깔면 스프링부트를 실행할 때, RabbitMQ 서버와 Connect한다.
![Pasted image 20250612160201.png](/img/user/supporter/image/Pasted%20image%2020250612160201.png)


## Config 

앞서 배운 Core Concept들을 Spring에서 어떻게 Config하는지 알아볼 것 
[[Basic **Concepts** Overview\|Basic **Concepts** Overview]]

### 기본 Queue-Exchange-Binding Config
```java 
@ConfigurationProperties(prefix = "rabbitmq")  
@Validated  
public record RabbitMQConfigProperties(  
  
    String queue_name,  
    String exchange_name,  
    String routing_key) {

---

@Configuration  
@RequiredArgsConstructor  
public class RabbitMQConfig {  
  
  private final RabbitMQConfigProperties rabbitMQProperties;  
  
  @Bean  
  public Queue queue() {  
    return new Queue(rabbitMQProperties.queue_name());  
  }  
  
  @Bean  
  public TopicExchange exchange() {  
    return new TopicExchange(rabbitMQProperties.exchange_name());  
  }  
  
  @Bean  
  public Binding binding() {  
    return BindingBuilder  
        .bind(queue())  
        .to(exchange())  
        .with(rabbitMQProperties.routing_key());  
  }
```



이 외에도 ConnectionFactory, RabbitTemplate, RabbitAdmin도 Config할 수 있지만 이 3개는 Auto-Config 되는 대상들이라 따로 설정하지 않아도 된다.



### Producer 생성 

우선 Message를 보내기 위해 RabbitTemplate을 만들어야한다
RabbitTemplate은 Auto-Config가 되어 있기에 거기에 내가 만든 Queue, Exchange, Binding등을 Inject해주면 된다.

```java
@Service  
@RequiredArgsConstructor  
@Slf4j  
public class RabbitMQProducer {  
  
  private final RabbitMQConfigProperties rabbitMQProperties;  
  private final RabbitTemplate rabbitTemplate;  
  
  public void sendMessage(String message) {  
    // exchange, routingKey, 메시지 template에 전달  
    log.info("보낼 메시지 : {}", message);  
    rabbitTemplate.convertAndSend(  
        rabbitMQProperties.exchange_name(),  
        rabbitMQProperties.routing_key(),  
        message  
    );
```

> RabbitMQTemplate에 exchange, routingKey, 메시지 를 전달하는 메서드

`convertAndSend( )`
- 파라미터 종류가 굉장히 다양하다
- 이번 예시에서 쓴 파라미터는  (exchange, routingKey, Object)이다
	- Queue에 메시지를 보내기 위해 exchange랑 routingKey정보가 필요하기 때문



> 컨트롤러로 보내고 확인해보기 


`localhost:15672 -> Queue and Streams -> Get Message(s)`

![Pasted image 20250613100232.png](/img/user/supporter/image/Pasted%20image%2020250613100232.png)
### Consumer 생성 
@RabbitListener 사용 

```java
@Slf4j  
@Service  
public class RabbitMQConsumer {  
      
    @RabbitListener(queues = {"${rabbitmq.queue.name}"}) // 이게 뭐지  
    public void consume(String message){  
        log.info("Message received -> {}", message);  
    }
```
