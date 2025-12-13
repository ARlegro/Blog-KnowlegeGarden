---
{"dg-publish":true,"permalink":"/DevStudy/Backend/RM/websocket_stomp/Spring Messaging/","noteIcon":"","created":"2025-12-03T14:52:49.971+09:00","updated":"2025-12-13T10:20:40.879+09:00"}
---



Spring WebSocket 의존성을 추가하면 Spring Messaging도 같이 포함되어 있다.
왜 이렇게 같이 포함되게 했을까❓


> 우선, Spring 메시징을 이해하기 위해서는 STOMP 프로토콜의 이해가 필요하다 

### 0.1.  STOMP 
Simple Text Oriented Messaging Protocol

> 메시지 브로커를 활용하여 쉽게 메시지를 주고 받을 수 있는 프로토콜
> Publisher - Message Broker - Subscriber

>[!tip] WebSocket 위에 얹어 함께 사용할 수 있는 하위(서브) 프로토콜이다. ⭐⭐



### 0.2.  왜 굳이 WebSocket에 STOMP를 ❓
>[!QUESTION] WebSocket에 STOMP를 더했을 때 이점은 뭘까 ❓


우선, WebSocket은 Text, Binary 타입을 양방향으로 주고 받을 수 있는 프로토콜이다.
근데 WebSocket은 이를 어떤 형식으로 전해줄지는 정해진게 없다.
근데 프로젝트를 하다보면 클라이언트와 주고받을 메시지 타입, 설정 정보들을 정해놓고 한다.
클라-서버는 이렇게 정의해서 로직을 짠다.
바로 여기서 도움을 주는게 STOMP이다

STOMP는 프레임 단위의 프로토콜 형식을 갖추고 있다.
- Command
- Header
- Body

![Pasted image 20250615194457.png](/img/user/supporter/image/Pasted%20image%2020250615194457.png)
```TEXT 
좌 = WebSocket 만 썼을 때 오고가는 메시지 
우 = WebSocket + STOMP 썼을 때 오고가는 메시지 
```

#### 0.2.1.  장점 정리
1. 컨벤션을 따로 정의할 필요가 없다.
2. 연결 주소마다 새로 핸들러를 구현하고 설정할 필요가 없다
3. 외부 Messaging Queue를 사용할 수 있다. ex. R.M
4. Spring Security 사용 가능 



### 0.3.  STOMP 동작 흐름 
https://docs.spring.io/spring-framework/reference/web/websocket/stomp/message-flow.html
![Pasted image 20250615194658.png](/img/user/supporter/image/Pasted%20image%2020250615194658.png)





## 1.  WebSocket + STOMP in Spring 

#EnableWebSocketMessageBroker 
#WebSocketMessageBrokerConfigurer

### 1.1.  기본 코드 
```JAVA 
@Configuration  
@EnableWebSocketMessageBroker  
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {  
  
    @Override  
    public void configureMessageBroker(MessageBrokerRegistry registry) {  
        // 클라이언트 구독 경로  
        registry.enableSimpleBroker("/topic");  
        // 서버 발행 경로  
        registry.setApplicationDestinationPrefixes("/app");  
    }  
  
    @Override  
    public void registerStompEndpoints(StompEndpointRegistry registry) {  
        registry.addEndpoint("/ws").setAllowedOriginPatterns("*").withSockJS();  
    }
```


#### 1.1.1.  메시지 브로커 설정 
> 메시지 브로커를 설정. 클라이언트가 구독하고, 서버가 메시지를 보내는 경로를 정한다.
```java
@Override  
public void configureMessageBroker(MessageBrokerRegistry registry) {  
    // 클라이언트 구독 경로  
    registry.enableSimpleBroker("/topic");  
    // 서버 발행 경로  
    registry.setApplicationDestinationPrefixes("/app");  
}

	// 스프링이 구현해 놓은 enableSimpleBroker
	public SimpleBrokerRegistration enableSimpleBroker(String... destinationPrefixes) {  
	  this.simpleBrokerRegistration = new SimpleBrokerRegistration(this.clientInboundChannel, this.clientOutboundChannel, destinationPrefixes);  
	  return this.simpleBrokerRegistration;  
	}
```
`enableSimpleBroker`
- 내장 브로커 사용하겠다는 설정(spring이 제공)
- 파라미터값 의미 : 클라이언트가 구독할 수 있는 경로를 의미 

`setApplicationDestinationPrefixes`
- 클라이언트가 메시지를 **서버로 보낼 때 사용하는 prefix**
- 서버 쪽에서 메시지를 수신할 Controller 메서드를 찾을 때 사용
```text
Client: SEND /app/chat/message
Server: @MessageMapping("/chat/message")
```
- 즉, `@MessageMapping("/chat/message")`와 `/app/chat/message`는 매핑됨.


#### 1.1.2.  webSocket 연결 시도 등록 

```java 
@Override  
public void registerStompEndpoints(StompEndpointRegistry registry) {  
    registry.addEndpoint("/ws")
				    .setAllowedOriginPatterns("*")
				    .withSockJS();  
}
```
 `withSockJS()`
- 브라우저가 WebSocket을 지원하지 않는 경우를 위한 **fallback**으로 **SockJS**를 사용.
- SockJS는 XHR-polling, iframe, etc. 방식으로 WebSocket을 흉내냄 
> 클라이언트에서는 자동으로 WebSocket 또는 SockJS 중 사용 가능한 것을 선택