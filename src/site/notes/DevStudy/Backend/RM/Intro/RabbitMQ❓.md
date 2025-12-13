---
{"dg-publish":true,"permalink":"/DevStudy/Backend/RM/Intro/RabbitMQ❓/","noteIcon":"","created":"2025-12-03T14:52:50.056+09:00","updated":"2025-12-13T10:21:43.390+09:00"}
---



### RabbitMQ란❓

> 오픈 소스 메시지 브로커

다양한 시스템, 구성요소간의 통신을 촉지하는 오픈 소스 메시지 브로커

**주요 특징과 역할** 
- **Message Queuing 시스템** 
	- 애플리케이션 구성 요소들 사이에서 메시지를 큐(Queue)에 저장하고 관리하며, 생산자(Producer)가 보낸 메시지를 소비자(Consumer)가 나중에 가져가 처리할 수 있도록 중개

- **비동기 통신**
	- 메시지 전송 후 즉각적인 **응답을 기다리지 않고** 다음 작업을 수행할 수 있어, 전체 시스템의 성능과 응답성을 향상시킨다. 
	  
- **미들웨어 역할**
	- 다른 애플레이션끼리 메시지를 주고 받게 해준다.
	  
- **다양한 프로토콜 지원**
	- AMQP(Advanced Message Queuing Protocol)를 기반으로 한다.
	- 하지만, 최근에는 MQTT, STOMP, HTTP 등 다양한 메시징 프로토콜을 지원하여 여러 기술 스택과의 통합이 용이
	
- **주요 활용 분야** : MSA, Event-Driven 시스템, 알림 서비스, 실시간 데이터 청리 등 




![Pasted image 20250612144016.png](/img/user/supporter/image/Pasted%20image%2020250612144016.png)



### 마이크로서비스에서 RabbitMQ의 사용 
마이크로서비스에서는 각 서비스마다 다른 애플리케이션에서 구동된다.
각 서비스가 연결되어 작동하는 경우가 있는데, 이 경우에 Message Broker가 중간에서 전달하는 역할을 한다.

![Pasted image 20250612144343.png](/img/user/supporter/image/Pasted%20image%2020250612144343.png)



### 아키텍쳐 

![Pasted image 20250612151549.png](/img/user/supporter/image/Pasted%20image%2020250612151549.png)

![Pasted image 20250612151703.png](/img/user/supporter/image/Pasted%20image%2020250612151703.png)

1. Producer는 Exchange에 메시지를 보낸다.
2. Exchange는 Routing Key를 활용해 메시지를 각 큐에 라우팅한다
3. Cousumer는 Queue에 있는 메시지를 읽는다.


