---
{"dg-publish":true,"permalink":"/프로젝트/나만무/RabbitMQ/나만무 - Rabbit MQ 도입/","noteIcon":"","created":"2025-12-06T12:52:49.761+09:00","updated":"2025-12-10T10:54:37.735+09:00"}
---



출처 : https://docs.nestjs.com/microservices/rabbitmq


설치 
```bash
npm i --save @nestjs/microservices
npm i --save amqplib amqp-connection-manager
```

- `@nestjs/microservices` : Nest 마이크로서비스 기능 제공
- `amqplib`, `amqp-connection-manager` : 실제 RabbitMQ 연결/채널 관리하는 라이브러리



## 1.  consumer 만들기  


### 1.1.  구독 
> nestjs의 App을 RabbitMQ와 연결하고 다양한 옵션들을 알아볼 것 

> [!WARNING] Produce 기능만 할거면 아래의 `connectMicroservice`는 필요 없다.

```JS
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);

  app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.RMQ,
    options: {
      /**
       * 어떤 RabbitMQ 서버에 붙을건지 정하는 옵션
       * 배열인 이유 : 여러 개의 브로커 주소를 넣으면 서버 죽을때 다른 곳으로 가면 돼서 고가용성이 높아짐
       * 형식 : 'ampq://유저:비번@호스트:포트'
       */
      urls: [process.env.RABBITMQ_URL ?? 'amqp://localhost:5672'],
      /**
       * 구독할 Queue의 이름(소비할 작업의 대기열 이름)
       * 이 이름의 Queue에 쌓인 메시지를 소비하겠다고 선언하는 것
       **/
      queue: 'matetrip_embedding_queue',
      /**
       * Queue 생성 시 기본 옵션
       * - durable = true : 브로커가 재시작되어도 Queue자체는 살아남기. 즉, RabbitMQ 서버가 꺼져도 큐를 디스크에 유지할지 여부
       * - noAck: false :
       */
      queueOptions: {
        durable: true,
      },

      /**
       * true: 메시지를 보낸 후 정상 처리 Ack를 기다리지 않는 것. 즉, 메시지 보내면 완료된 걸로 처리
       * false
       *  - 직접 channel.ack(message)를 호출해야 메시지가 진짜로 제거됨.
       *  - Consumer가 크래시 나면 → RabbitMQ가 메시지를 다시 큐에 되돌리고, 다른 Consumer에게 재전송 가능
       *      장점: 안정성 (작업 보장)
       *      단점: 코드에 Ack 처리 추가 필요, 약간 더 복잡
       * 크롤링 같은 작업 or 임베딩 작업은 프로세스가 죽으면 다시 실행되야 하므로 noAck: false가 맞음
       */
      noAck: false,
      /**
       * 한 Consumer가 한 번에 미리 받아둘 메시지 개수
       * 1, 10
       * 동시에 여러 작업을 진행하는 워커와 궁합이 좋음
       * 테스크의 자원/부하를 체크하고 조절 (초기 : 5 ~ 20)
       */
      prefetchCount: 10,
    },
```


## 2.  Producer 만들기 

> NestJS 백엔드 ➡ RabbitMQ로 메시지 보내는 작업

단계 
1. AppModule에 RabbitMQ Client 등록
2. Producer용 Service 하나 만들기
3. 실제 도메인 서비스에서 사용하기

### 2.1.  AppModule에 RabbitMQ Client 등록

```js
@Module({
  imports:화
- Channel❓
	- **Connection 안에서 만들어지는 논리적 통신 통로**
	- 하나의 Connection에 여러 개의 Channel 가능 
	- 참고 : NestJS는 `@nestjs/microservices`가 내부에서 자동으로 만들어서 관리해줘서 따로 설정이 필요없다.

> [!INFO] NestJS 내부 
> `Transport.RMQ`를 쓰면 Nest가 내부적으로 이런 흐름을 탄다
> 1.  `urls` 기반으로 **amqp connection 생성**
> 2. 그 connection 위에 **channel 생성**
> 3. `queue`, `queueOptions` 가지고 **channel.queueDeclare(...) 실행**
> 4. 그걸 감싼 **ClientProxy 인스턴스**를 만들어서  `@Inject('EMBEDDING_CLIENT')` 로 주입해줌

