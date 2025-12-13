---
{"dg-publish":true,"permalink":"/DevStudy/Backend/RM/Intro/MessageListnerAdapter/","noteIcon":"","created":"2025-12-03T14:52:50.047+09:00","updated":"2025-12-13T10:21:40.325+09:00"}
---



> Spring AMQP 프레임워크에서 사용되는 **어댑터(Adapter) 디자인 패턴**을 구현한 클래스


Spring AMQP의 `SimpleMessageListenerContainer`는 `MessageListenr`인터페이스를 구현한 객체를 받아서 메시지를 처리한다.
이 인터페이스는 onMessage(Message message)와 같은 특정 형태의 메서드가 정의되어 있다.
하지만 RabbitMQ를 사용하는 개발자는 메시지 처리 로직 구현 시 자신의 POJO객체에 `handlePayment(String content)`같은 메서드를 사용한다.
이렇게 유연한 메서드를 사용할 수 있게 해주는 것이 MessageListenerAdaper이다.


![Pasted image 20250615105540.png](/img/user/supporter/image/Pasted%20image%2020250615105540.png)
- **`MessageListenerAdapter`는 개발자가 만든 일반 POJO 객체의 메소드를 받아서, 그것을 `MessageListener` 인터페이스의 `onMessage` 메소드처럼 작동하도록 "적응(adapt)"시켜 줍니다.**
- 즉, `SimpleMessageListenerContainer`는 `MessageListenerAdapter`를 통해서 POJO 메소드를 호출하게 되는 것



**`@RabbitListener` 어노테이션의 역할**:
아래와 같은 내부적인 작업을 수행
- `SimpleRabbitListenerContainerFactory`를 사용하여 **`SimpleMessageListenerContainer` 인스턴스를 생성** 
	- Note : `SimpleRabbitListenerContainerFactory`는 스프링 부트가 Auto-Config한 빈으로 이름(factory) 그대로 container를 생성하는 역할 
- 이때, `@RabbitListener`가 붙은 메소드를 `MessageListenerAdapter`를 통해 `SimpleMessageListenerContainer`에 등록

> 즉 @RabbitListenr는 고수준의 추상화 기술 (리스너 컨테이너 인스턴스 생성 + 컨테이너에 adaper 등록)




