---
{"dg-publish":true,"permalink":"/DevStudy/Backend/RM/Intro/Send JSON And User Multiple Queue in RM/","noteIcon":"","created":"2025-12-03T14:52:50.018+09:00","updated":"2025-12-13T10:21:55.588+09:00"}
---



> 알아볼 것 
> 1. JSON 타입으로 메시지 보내기
> 2. Multiple Queue이용하기 

![Pasted image 20250612171312.png](/img/user/supporter/image/Pasted%20image%2020250612171312.png)



일단 직렬화/역직렬화할 수 있는 POJO객체가 필요하다 



### 0.1.  Message Converter, Template 커스텀 
> JSON을 직렬화 역직렬화 할 수 있게 등록 


>[!EXAMPLE] REST API에서와 달리 따로 Json용 MesssageConverter가 필요한 이유 
>- Spring AMQP에서 `RabbitTemplate`은 내부적으로 `MessageConverter`를 이용해서 객체를 byte[]로 변환해 메시지를 보낸다.  
>- 기본값은 `SimpleMessageConverter`인데, 이건 JSON을 다루지 않는다.
>	- simpleMessageConverter는 String, byte[], Serializable 객체만 처리 가능 

기본 RabbitTemplate의 MessageConverter
```java 
public class RabbitTemplate extends RabbitAccessor ... {

		private MessageConverter messageConverter = new SimpleMessageConverter();
}
```


**✅Message Converter 빈 등록 - amqp 버전** 
```java 
import org.springframework.amqp.support.converter.MessageConverter;

@Bean  
public MessageConverter converter() {  
		return new Jackson2JsonMessageConverter();  
}
```

> Note : MessageConverter는 amqp버전으로 import해야한다!!! 


**✅RabbitTemplate 커스텀**
```java
@Bean  
public AmqpTemplate amqpTemplate(ConnectionFactory connectionFactory) {  
    RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);  
    rabbitTemplate.setMessageConverter(converter());  
    return rabbitTemplate;  
}
```
- AmqpTemplate : 인터페이스
- RabbitTemplate : 구현체
- ConnectionFactory도 amqp 버전 

### 0.2.  Queue-Binding 재설정 

✅JSON 전용 Queue 생성
```JAVA 
@Bean  
public Queue jsonQueue() {  
  return new Queue(rabbitMQProperties.queue_name_json());  
}

@Bean  
public Binding jsonBinding() {  
  return BindingBuilder  
      .bind(jsonQueue())  
      .to(exchange())  
      .with(rabbitMQProperties.routing_key_json());  
```


### 0.3.  Producer-Consumer 
```JAVA
public void sendJsonMessage(CustomerDTO customerDTO) {  
  log.info("보낼 JSON : {} ", customerDTO);  
  rabbitTemplate.convertAndSend(  
      rabbitMQProperties.exchange_name(),  
      rabbitMQProperties.routing_key_json(),  
      customerDTO  
  );

@RabbitListener(queues = {"${rabbitmq.queue-name-json}"})  
public void consumeJsonMessage(CustomerDTO message) {  
  log.info("받은 JSON 메시지 : {}", message);  
}


## JSON 으로 메시지 해보기 


```

### 0.4.  테스트 


컨트롤러 
```JAVA 
@PostMapping("/publish")  
public ResponseEntity<String> sendJsonMessage(@RequestBody @Validated CustomerDTO message) {  
		rabbitMQProducer.sendJsonMessage(message);  
		return ResponseEntity.status(HttpStatus.OK).body("JSON-Message sent to R.M");  
}  
```
```JSON
{
    "id": 1,
    "name" : "yohan"
}
```

```bash
[rabbitmq-study] [nio-8080-exec-4] s.rabbitmq.producer.RabbitMQProducer     
: 보낼 JSON : CustomerDTO[id=1, name=yohan] 
[rabbitmq-study] [ntContainer#1-1] s.rabbitmq.consumer.RabbitMQConsumer     
: 받은 JSON 메시지 : CustomerDTO[id=1, name=yohan]
```


