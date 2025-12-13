---
{"dg-publish":true,"permalink":"/DevStudy/Backend/RM/Intro/Basic Concepts Overview(RM)/","noteIcon":"","created":"2025-12-03T14:52:50.038+09:00","updated":"2025-12-13T10:21:35.948+09:00"}
---

#Producer #Consumer #Queue #Exchange #Binding #Routing_Key #Message




![Pasted image 20250615100304.png](/img/user/supporter/image/Pasted%20image%2020250615100304.png)

![Pasted image 20250612144016.png](/img/user/supporter/image/Pasted%20image%2020250612144016.png)




| 용어                      | 내용                                                                           |
| ----------------------- | ---------------------------------------------------------------------------- |
| Producer(생산자)           | 메시지를 생성하고 Consumer에게 보내기 위해 메시지 브로커를 통해 "간접적"으로 전송하는 구성요소이다                  |
| Message(메시지)            | Producer가 Cosumer에게 전달하려는 실제 데이터                                             |
| Message Broker(메시지 브로커) | 메시지를 생성하고 소비하는 클라이언트 간에 메시지를 전달하는 미들웨어<br>ex. Kafka, RabbitMQ 등              |
| Exchange(교환기)           | 메시지를 수신하고 처리할 대상을 결정하는 구성요소                                                  |
| Consumer(소비자)           | 메시지 브로커에서 메시지를 가져와서 ‘수신’하고 ‘처리’를 하는 주체                                       |
| Binding(바인딩)            | Queue와 Exchange 사이의 논리적인 연결 관계                                               |
| Queue                   | 메시지 브로커 내에 메시지를 저장하는 저장소 or 버퍼 <br>송신자가 메시지를 전송하면 이를 수신자가 처리할때까지 대기시킵니다      |
| Routing Key             | 메시지를 어떤 Queue에 라우팅할지 결정하기 위해 사용된다<br>Producer가 메시지를 보낼 때 Routing Key와 함께 보낸다 |
### 0.1.  Producer 

> [!abstract] 메시지를 생성하고 RabbitMQ 브로커로 전송하는 애플리케이션 또는 서비스

1.  Consumer에게 보낼 **메시지를 생성**하고 
2. 중간 매체들을(ex. Exchange 메시지 브로커)를 통해 **"간접적"으로 전송하는 구성요소**이다
	- Consumer에게 직접 보내는 것이 아닌 항상 Exchange를 통해 보낸다.
		- **느슨한 결합** : Producer는 단순히 Exchange만 알면 되고 최종 목적지인 Queue에 대해 알지 않아도 되기 때문에 시스템간 유연성을 높일 수 있다.
		  
### 0.2.  Consumer 
> [!abstract] RabbitMQ 브로커로부터 메시지를 수신하여 처리하는 애플리케이션 또는 서비스

- Consumer는 메시지를 수신하기 위해 특정 **Queue에 연결**하고, 큐에 도착하는 **메시지를 읽어온다**
- **다중 소비자 가능** : 하나의 Queue에 여러 Consumer가 연결될 수 있다.

### 0.3.  Queue

> [!abstract] Message Broker 내에서 메시지를 임시로 저장하는 저장소 or 버퍼 

- **역할**
	1. **메시지 대기열** : 메시지들이 소비자에게 전달되기 전에 순서를 지켜 대기하는 곳
		- 즉, Producer로부터 전달된 메시지가 바로 소비되는 것은 아니라, Consumer가 가져갈 때까지 큐에 대기한다.

	2. **버퍼링** : 소비자의 처리 속도에 맞춰 메시지를 저장한다. 이로 인해 병목 현상을 완화 
	3. **FIFO** : 메시지는 큐에 들어온 순서대로 소비된다.

```JAVA 
@Bean  
public Queue queue() {  
    return new Queue(queueName, false);   << 2번째 파라미터 = durable
}
```


### 0.4.  메시지(Message)

> [!abstract] Producer가 Cosumer에게 전달하려는 실제 데이터 

- **전달 형식** : 일반적으로 **바이트 스트림(Byte Stream)** 형태로 전달됩니다. 즉, 단순히 텍스트뿐만 아니라 JSON, XML, 이진 데이터 등 다양한 형식의 데이터를 담을 수 있다.
- **구성**
	1. **Header (속성)** : 메시지의 메타데이터를 포함 ex. 메시지 타입, 인코딩 방식, 상관관계 id 등  
	2. **Payload(Body)** : 메시지가 실제로 전달하고자 하는 데이터 내용

### 0.5.  메시지 브로커 
> [!abstract] 메시지를 주고받는 것을 **중개하는 소프트웨어** 
> - 메시지 지향 미들웨어의 핵심 구성 요소 
> - 서로 다른 시스템들이 직접 통신하지 않고, 대신 브로커를 통해 간접적으로 비동기 통신을하도록 돕는다.


### 0.6.  Exchange ⭐⭐

> [!abstract] 정의 
> - **메시지를 수신**하고 **처리할 대상을 결정**하는 구성요소
> - **라우팅 에이전트 역할** : Producer와 Queue의 중개자 역할을 한다

✔**주요 역할** 
1. **메시지 수신** : Producer로부터 메시지를 받는다.
2. **라우팅** : 메시지를 어떤 큐로 보낼지 결정
3. **바인딩** : Queue와의 연결관계를 형성(binding)하여 이를 기반으로 라우팅 결정을 내린다.


**✅Exchange 타입 ⭐**

> 메시지를 라우팅하는 방식에 따라 여러 타입으로 나뉜다.

아래는 나중에 Develop하기

| Exchange 유형                   | 설명                                                                                 |
| ----------------------------- | ---------------------------------------------------------------------------------- |
| **RabbitMQ Direct Exchange**  | `Routing Key`와 `Binding Key`가 정확한 일치하는 메시지만 해당 큐로 전달                               |
| **RabbitMQ Fanout Exchange**  | 해당 Exchange에 **바인딩 된 모든 큐로 메시지를 복사하여 전달** <br>`Routing Key`는 전부 무시된다.              |
| **RabbitMQ Topic Exchange**   | `Routing Key`를 **패턴 매칭**하여 메시지를 전달<br>와일드 카드 ex. `* #`<br>유연한 발행-구독 시스템에서 자주 활용된다. |
| **RabbitMQ Headers Exchange** | `Routing Key` 대신 메시지 헤더의 속성들을 기반으로 라우팅                                             |



### 0.7.  Binding(바인딩)

> [!abstract] 정의 
> - **Queue와 Exchange 사이의 논리적인 연결 관계**를 의미 
> - 이 연결은 메시지가 Exchange로부터 어떤 Queue로 전달될 수 있는지를 정의하는 규칙이다.

```java 
@Bean
public Binding binding() {
		return BindingBuilder.bind(queue())
								.to(exchange())
								.with("routing_key")  // 바인딩 키 
```

**✔역할**
- Exchange가 메시지를 라우팅할 때, 이 **바인딩 관계를 통해 메시지를 특정 큐로 보낼지 말지를 결정**

**✅바인딩 키** (라우팅 키와 매칭)
- 바인딩을 생성 시, Exchange와 Queue를 연결하는 데 사용되는 특정 문자열 또는 패턴
- **메시지의 Routing Key와 일치하는 Queue로 메시지가 라우팅**된다 ❗❗


### 0.8.  Routing Key(라우팅 키)
- 정의 : Producer가 메시지를 Exchange로 보낼 때 메시지와 함께 포함하는 문자열 속성
- 역할 : Exchange가 이 key를 보고 **어떤 Queue로 라우팅할지 결정**하게 해준다.
- 라우팅 키의 해석 방식은 Exchange 타입에 따라 달라진다.





### 0.9.  몇 개 팁 

#### 0.9.1.  하나의 Queue에 여러 Consumer가 있을 수 있다.
![Pasted image 20250612144759.png](/img/user/supporter/image/Pasted%20image%2020250612144759.png)



#### 0.9.2.  Exchange 개념 다시보기 

 >메시지를 수신하고 처리할 대상을 결정하는 역할 (수신 + 라우팅 에이전트)



- **Producer와 Queue의 중개자 역할**
	- Producer가 메시지를 RabbitMQ로 보낼 때, 메시지는 Queue에 직접 전달되는 것이 아니라 반드시 **Exchange**로 먼저 전달된ㄷ다.
	- Exchange는 이 메시지를 받아서 자신의 타입과 메시지에 포함된 라우팅 키(routing key)에 따라 하나 이상의 큐로 메시지를 전달
- **주요 역할**
	- 메시지 수신 : 생산자로부터 메시지를 받는다.
	- 라우팅 : 메시지를 어떤 큐로 보낼지 결정 
	- 바인딩 : 큐사이의 연결관계를 정의한다.
	  
![Pasted image 20250612145106.png](/img/user/supporter/image/Pasted%20image%2020250612145106.png)



https://adjh54.tistory.com/497#2.%20Exchange%20%EC%9C%A0%ED%98%95-1-2

### 0.10.  Exchange 유형에 따른 처리 흐름 
![Pasted image 20250615101252.png](/img/user/supporter/image/Pasted%20image%2020250615101252.png)

## 1.  추가 용어 

### 1.1.  Prefetch Count 
- 소비자가 받을 수 있는 최대 메시지 수 (설정)
- 설정 이유 : 한 번에 많은 양의 메시지를 처리하지 않도록 하여 소비자 성능 최적화 


### 1.2.  Virtual Host(가상 호스트)
- RabbitMQ 서버 내의 논리적 구획으로,
- Queue, Exchange, 사용자 권한 등을 구분 
- 하나의 RabbitMQ 서버 내에 여러 개의 가상 호스트를 설정 가능 

### 1.3.  Dead Letter Queue(DLQ)
- 메시지가 처리되지 못하거나 유효 기간이 지난 경우 별도의 큐로 이동하는 구조로 설정 가능 


### 1.4.  RabbitTemplate 

- ConnectionFactory는 RabbitMQ와의 연결을 관리하는 객체로, rabbitTemplate이 메시지를 전송할 때 사용할 연결을 제공한다.