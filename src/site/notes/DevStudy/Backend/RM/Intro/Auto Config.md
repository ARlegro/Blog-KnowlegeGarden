---
{"dg-publish":true,"permalink":"/DevStudy/Backend/RM/Intro/Auto Config/","noteIcon":"","created":"2025-12-03T14:52:50.009+09:00","updated":"2025-12-13T10:21:30.155+09:00"}
---


Spring boot에서 gradle설정 시 RabbitMQ의 어떤 것들이 자동으로 config 될까?

### Spring Boot의 자동 설정 (Auto-configuration)

Spring Boot는 RabbitMQ와의 통합을 매우 간소화하여 제공합니다.
Spring Boot는 클래스패스에 RabbitMQ 관련 라이브러리가 있음을 감지하고, 필요한 빈(Bean)들을 자동으로 구성해줍니다.

이 자동 구성에는 다음과 같은 것들이 포함됩니다:
1. **`ConnectionFactory`**: RabbitMQ 서버와의 연결을 관리하는 팩토리.
2. **`RabbitTemplate`**: 메시지 전송(생산자 역할)을 위한 템플릿.
3. **`RabbitAdmin`**: 큐, 교환기, 바인딩 등을 관리하는 어드민 객체.
4. **`SimpleRabbitListenerContainerFactory`**: `@RabbitListener` 어노테이션이 붙은 메소드를 감지하고, 해당 메소드를 메시지 리스너로 등록하기 위한 `SimpleMessageListenerContainer` 인스턴스를 생성하고 구성하는 팩토리.

### `@RabbitListener` 어노테이션과 자동 설정의 연동

가장 일반적인 시나리오는 개발자가 직접 `SimpleMessageListenerContainer` 빈을 선언하는 대신, 메시지를 수신할 메소드에 `@RabbitListener` 어노테이션을 사용하는 것

```java
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@Component // Spring Bean으로 등록
public class MyMessageConsumer {

    @RabbitListener(queues = "myQueue") // "myQueue" 큐에서 메시지를 수신합니다.
    public void receiveMessage(String message) {
        System.out.println("Received message: " + message);
        // 메시지 처리 로직
    }
}
```

위 코드처럼 `@RabbitListener` 어노테이션만 사용하면, Spring Boot는 다음을 자동으로 처리합니다:
1. **`myQueue`라는 이름의 큐가 없으면 자동으로 생성** (물론 명시적으로 큐를 선언하여 관리하는 것이 좋습니다).
2. 이 리스너 메소드를 처리하기 위한 **`SimpleMessageListenerContainer` 인스턴스를 내부적으로 생성하고 구성**합니다.
3. 생성된 컨테이너가 `myQueue`에서 메시지를 가져와 `receiveMessage` 메소드로 전달하도록 연결합니다.
4. 컨테이너의 스레드 수, 재연결 로직, 오류 처리 등은 **Spring Boot의 기본 설정**에 따라 자동으로 적용됩니다.

### 자동 설정의 커스터마이징

대부분의 경우 Spring Boot의 기본 설정으로 충분하지만, 특정 요구사항이 있다면 자동 설정을 커스터마이징할 수 있습니다.

1. **`application.properties` 또는 `application.yml` 파일 사용:**
    - `spring.rabbitmq.listener.simple.concurrency=5`: 메시지 처리에 사용할 최소 스레드 수.
    - `spring.rabbitmq.listener.simple.max-concurrency=10`: 메시지 처리에 사용할 최대 스레드 수.
    - `spring.rabbitmq.listener.simple.acknowledge-mode=MANUAL`: 메시지 승인 모드 (기본값은 AUTO).
    - `spring.rabbitmq.listener.simple.auto-startup=true`: 컨테이너 자동 시작 여부.
    - 기타 다양한 옵션을 설정할 수 있습니다.
      
2. **`SimpleRabbitListenerContainerFactory` 빈 직접 정의:**
    - 더 세밀한 제어가 필요하거나 여러 종류의 `@RabbitListener`를 다른 설정으로 운영하고 싶을 때 사용합니다.
    - 이 팩토리를 정의하면 `@RabbitListener`는 기본 `SimpleRabbitListenerContainerFactory` 대신 사용자가 정의한 팩토리를 사용하게 됩니다.
  
    ```Java
    import org.springframework.amqp.rabbit.config.SimpleRabbitListenerContainerFactory;
    import org.springframework.amqp.rabbit.connection.ConnectionFactory;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    
    @Configuration
    public class RabbitListenerConfig {
    
        @Bean
        public SimpleRabbitListenerContainerFactory myCustomContainerFactory(ConnectionFactory connectionFactory) {
            SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
            factory.setConnectionFactory(connectionFactory);
            factory.setConcurrentConsumers(3); // 커스텀 스레드 수
            factory.setMaxConcurrentConsumers(5);
            factory.setPrefetchCount(1); // 한 번에 가져올 메시지 수
            // 기타 다양한 설정 가능
            return factory;
        }
    
        // @RabbitListener(queues = "myCustomQueue", containerFactory = "myCustomContainerFactory")
        // public void receiveCustomMessage(String message) {
        //     System.out.println("Received custom message: " + message);
        // }
    }
    ```
    

### 결론

최신 Spring Boot에서는 `SimpleMessageListenerContainer`의 복잡한 설정 과정을 개발자가 직접 할 필요가 거의 없습니다. `@RabbitListener` 어노테이션만 사용하면 Spring Boot가 자동으로 적절한 `SimpleMessageListenerContainer`를 구성하여 메시지 수신을 가능하게 해줍니다. 이는 개발 생산성을 크게 향상시키고 RabbitMQ를 통한 비동기 메시징 시스템 구축을 훨씬 쉽게 만듭니다.

