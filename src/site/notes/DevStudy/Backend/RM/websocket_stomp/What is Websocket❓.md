---
{"dg-publish":true,"permalink":"/DevStudy/Backend/RM/websocket_stomp/What is Websocket❓/","noteIcon":"","created":"2025-12-03T14:52:49.981+09:00","updated":"2025-12-13T10:20:46.233+09:00"}
---


- websocket이 필요한 이유 
- HTTP vs 웹소켓 
- websocket이란?
- 장점 

### 0.1.  WebSocket 이란❓

#### 0.1.1.  개념 
- **양방향 통신**을 가능하게 하는 표준 프로토콜 
- 클라이언트-서버 간에 실시간 데이터를 주고받는 데 적합하다
- **데이터 전송 단위** : 프레임(Frame)
- **오버헤드가 낮다.**
- 연결이 초기화git rebase -i ccaf5493^된 후에는 상태 유지하며 데이터 교환 가능 

#### 0.1.2.  특징 
- **실시간성** : 실시간으로 데이터 주고 받을 수 있다.
- **효율성** 
	- HTTP Polling 방식보다 네트워크 자원과 대역폭 절약 가능 
	- HTTP 기본 통신 방법보다 연결 비용, 보내는 양이 저렴 
	  
- **양방향 통신(Full-Duplex)**
- **낮은 지연 시간** 
	- 한 번 연결되면 데이터 전송 속도가 매우 빠름 
	  
- **상태 유지** 


### 0.2.  HTTP vs WebSocket
#stateful

> websocket은 실시간성을 보장한다.
> - 실시간 보장 서비스 ➡ 게임, 채팅, 실시간 주식 거래 사이트 


하지만, HTTP도 실시간성을 보장하는 듯한 기법이 있다. ex) Polling, Long Polling, Steaming 

>[!QUESTION] 그렇다면 왜 실시간 통신에서 보통 WebSocket을 사용할까❓

|              | **HTTP**                         | **WebSocket**                                        |
| ------------ | -------------------------------- | ---------------------------------------------------- |
| **지향**       | 비 연결성                            | 연결 지향                                                |
| **연결 방식**    | 매번 연결을 맺고 끊는 비용 발생               | 한 번 연결 맺은 뒤 유지                                       |
| **통신 방식**    | 요청 - 응답  구조                      | 양방향 통신                                               |
| **보내야 하는 양** | WebSocket보다 프로토콜에서 보내야 하는 양이 많다. | 첫 연결 빼고는 정보를 주고받는 양이 적다.<br>(그래도 첫 연결은 TCP/IP H.S 必) |
WebSocket은 아래와 같은 장점이 있다(vs HTTP)
1. 매번 연결을 하고 끊는 비용 감소 
2. 양방향 통신 
3. 주고받는 양의 이점 -> 통신 비용 이점 

> 따라서, 실시간 양방향 통신에서는 WebSocket을 많이 쓴다.


### 0.3.  WebSocket을 지원하지 않는 환경이라면 ❓

> 대부분 WebSocket을 지원하지만, 전부 지원하지는 않는다.

#### 0.3.1.  지원하지 않는 환경 예시
1. 구 브라우저  ex. Internet Explorer 9, Safari 구버전, Android 구버전 
2. 특정 인앱 브라우저  ex. 카톡 내 브라우저, 인스타 내 브라우저
3. 네트워크 제약 있는 곳 ex. 회사/학교
4. 임베디드 환경  ex. 스마트 TV, 저사양 태블릿 


#### 0.3.2.  해결법 : 라이브러리(SockJS, Socket.IO)

> 이 라이브러리를 사용하면 WebSocket을 지원하지 않는 브라우져에서도 WebSocket같은 기능 구현 가능

Spring은 SockJS방식을 지원하며, WebSocket이 지원되지 않는 환경에서는 Polling or Streaming처럼 제공된다.

## 1.  WebSocket in Spirng 


### 1.1.  기본 
#WebSocketConfigurer  #EnableWebSocket
```java 
@Configuration  
@EnableWebSocket  
public class WebSocketConfig2 implements WebSocketConfigurer {  
  
  @Override  
  public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {  
      registry.addHandler(new WebSocketHandler(), "/ws-connect")  
			    .setAllowedOriginPatterns("*")  
			    .withSockJS();
  }
```
- `/ws-connect` : 웹소켓 연결 주소 
- `WebSocketHandler` : 직접 구현한 webSocket 핸들러 


### 1.2.  WebSocket핸들러 구현 예시
```java 
public class WebSocketHandler extends BinaryWebSocketHandler {  
  
  private final Set<WebSocketSession> sessions = ConcurrentHashMap.newKeySet();  
  
  @Override  
  public void afterConnectionEstablished(WebSocketSession session) throws Exception {  
    sessions.add(session);  
  }  
  
  @Override  
  protected void handleTextMessage(WebSocketSession session, TextMessage message) {  
    String payload = message.getPayload();  
  
    for (WebSocketSession s : sessions) {  
      try {  
        s.sendMessage(new TextMessage(payload));  
      } catch (IOException e) {  
        throw new RuntimeException(e);  
      }  
    }  
  }  
  
  @Override  
  public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {  
    sessions.remove(session);  
  }
```


#### 1.2.1.  상속 
우선 WebSocket은 Text, Binary 타입을 지원한다.
따라서, 필요에 따라서 알아서 상속하면 된다.
```java
// 1. Text
public class WebSocketHandler extends TextWebSocketHandler {

// 2. Binary
public class WebSocketHandler extends BinaryWebSocketHandler {
```

#### 1.2.2.  afterConnectionEstablished(closed) 
```java
// 커넥션이 생겼을 때 
@Override  
public void afterConnectionEstablished(WebSocketSession session) throws Exception {  
  sessions.add(session);  
}
```

오버라이드한 메서드이다.
파라미터에 WebSocketSession이 있는데 이는 HTTP Session과 다르다.
`WebSocketSession` = webSocket 연결 시 연결 정보를 담고 있는 객체 
반면 HTTP session은 ?

커넥션이 맺어질 때 webSocket session을 추가하는 것 

```java
// 커넥션이 끊겼을 때
@Override  
public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {  
	  sessions.remove(session);  
}
```


>[!QUESTION] 위의 핸들러에서 왜 직접 Session을 관리하게 구현했을까❓ 
>- 이렇게 관리해야 모든 세션에 동시에 메시지 전달같은 관리 기능이 가능하기 때문이다.




[[DevStudy/Backend/RM/websocket_stomp/Spring Messaging\|Spring Messaging]]