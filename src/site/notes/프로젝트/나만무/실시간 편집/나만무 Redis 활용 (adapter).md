---
{"dg-publish":true,"permalink":"/프로젝트/나만무/실시간 편집/나만무 Redis 활용 (adapter)/","noteIcon":"","created":"2025-12-03T14:53:06.776+09:00","updated":"2025-12-10T13:56:20.608+09:00"}
---


---
## 1.  세팅 
1. *Redis 컨테이너*
	```bash
	docker run --name redis -p 6379:6379 -d redis
	```

`npm i --save @nestjs/websockets @nestjs/platform-socket.io socket.io`


1. **npm 설치**
	```bash
	npm i redis ioredis @socket.io/redis-adapter @nestjs/platform-socket.io @socket.io/redis-emitter
	```

2. socket 설치
```js
npm i --save @nestjs/websockets @nestjs/platform-socket.io socket.io
```

---
### 1.1.  redis 도커 띄우기 

```yaml
# 레디스
services:
  redis_container:
    image: redis:latest
    container_name: redis-test
    ports:
      - '6379:6379'
  
    volumes:
      - redis_data:/usr/local/etc/redis/redis.conf
  
volumes:
  redis_data:
```


*✅테스트* 
```bash
docker exec -it redis-test redis-cli

SET watermelon 15000
GET watermelon
```

---
## 2.  개념 

---
### 2.1.  Pub/Sub 디자인 패턴 
> 이벤트 드리븐 아키텍쳐를 적용하는 한 가지 방법 

>[!QUESTION] What is Event Driven Architecture
>시스템이 이벤트나 상태 변화에 비동기적으로 대응되도록 설계된 아키텍쳐 


*Pub/Sub 패턴*
1. 이벤트 생성자가 이벤트 발행
2. 이벤트 소비자가 구독하여 이벤트를 수신하고 처리 

---
### 2.2.  adapter for socket
 


---
## 3.  Redis Adapter + Pub/Sub을 쓴 이유
> 실시간 UI 동기화 : 실시간 협업 시 멀티 인스턴스 환경에서 실시간 동기화를 유지하려고. 

---
### 3.1.  기본 Socket.IO의 한계 
>[!danger] 확장성의 한계 
>멀티 인스턴스 환경에서 실시간 동기화 문제가 발생할 수 있다. 

- 기본 Adapter는 프로세스 메모리만 쓴다.
- 따라서, 프로세스 안에서만 브로드 캐스트하기 때문에 Scale-Out을 통해 다른 인스턴스

---
### 3.2.  Redis Adapter + Pub/Sub을 쓴다면 

각 서버 인스턴스는 중앙 Message Broker인 Redis를 통해 pub/sub을 구현할 수 있다.
![Pasted image 20251105024010.png](/img/user/supporter/image/Pasted%20image%2020251105024010.png)

*이점*
1. **멀티 인스턴스 간 실시간 동기화**
    - 유저들이 어느 서버 인스턴스에 붙어 있든, 같은 워크스페이스/룸에 있으면 동일한 마킹 이벤트를 실시간으로 받음
2. **수평 확장(Scale-out) 가능**
    - 유저가 늘어나면 서버 인스턴스 수만 늘리면 됨
    - Redis가 중앙 허브 역할을 하기 때문에 방 정보/이벤트가 서버마다 따로 놀지 않음


> 즉, 수평 확장해도 실시간 동기화 이슈가 발생하지 않기 위해 

---
## 4.  redisAdpater 

> 기존 소켓 Adapter를 갈아끼우는 것 


![Pasted image 20251103154735.png](/img/user/supporter/image/Pasted%20image%2020251103154735.png)

---
### 4.1.  기존 NestJS IoAdapter 역할 

> Nest가 내부적으로 Socket.IO 서버를 생성하고 Gateway에 주입하는 팩토리 역할 


```js
export declare class IoAdapter extends AbstractWsAdapter {

    create(port: number, options?: ServerOptions & {
        namespace?: string;
        server?: any;
    }): Server;
    createIOServer(port: number, options?: any): any;
    ....
    close(server: Server): Promise<void>;
}
```

---
### 4.2.  RedisAdapter로 교체 후 등록

```js
export class RedisIoAdapter extends IoAdapter {
  private adapterConstructor: ReturnType<typeof createAdapter>;

  async connectToRedis(): Promise<void> {
	  // RedisClient를 생성 
    const pubClient = createClient({ url: `redis://localhost:6379` });
    // 동일한 설정의 Subscriber클라이언트 생성
    const subClient = pubClient.duplicate();

		// 2개의 Redis Client를 동시에 연결시키는 비동기 병렬 처리 
    await Promise.all([pubClient.connect(), subClient.connect()]);

		// Redis를 사용하는 Socket.IO 어댑터 생성(나중에 CreateIOServer()에서 사용)
    this.adapterConstructor = createAdapter(pubClient, subClient);
  }

  createIOServer(port: number, options?: ServerOptions): any {
    const server = super.createIOServer(port, options);
    server.adapter(this.adapterConstructor);
    return server;
  }
```

1. `RedisIoAdapter extends IoAdapter`
	- Socket.IO용 Redis 확장 어뎁터를 만드는 것 
	- `IoAdapter` = NestJS의 기본 WebSocket 어댑터 
	  
2. `private adapterConstructor: ReturnType<typeof createAdapter`
	- 어뎁터 인스턴스 생성기 
	- 이 생성기는 Redis pub/sub 클라이언트 쌍을 받아서 Socket.IO 서버 간 메시지를 공유하게 함
   
3. `connectToRedis`
	- 
	  
4. `createIOServer`
	- 반환 : Socket.IO 서버 인스턴스
	- create 함수에서 서버를 직접 구동(listen)하지는 않는다.
	- 이걸 반환하면 NestJS가 내부적으로 `listen()`처리 함 
	- **NestJS가 자동으로 호출**


등록 
```js

const app = await NestFactory.create(AppModule);
const redisIoAdapter = new RedisIoAdapter(app);
await redisIoAdapter.connectToRedis();

app.useWebSocketAdapter(redisIoAdapter);
```


>[!QUESTION] createIOServer() 직접 호출 없이도 Socket.IO 서버가 생성되는 이유 
>NestJS의 WebSocket 시스템은 **Gateway를 부트스트랩할 때 내부적으로 IoAdapter의 `createIOServer()`를 자동 호출**<BR>
>Nest의 애플리케이션 초기화 과정
>1. `SocketServerProvider`가 각 Gateway 클래스를 스캔
>2. `RedisIoAdapter.createIOServer()`를 호출해 Socket.IO 서버 생성
>3. 그 결과를 `@WebSocketServer()` 프로퍼티(`server`)에 주입
>4. 그리고 자동으로 `listen()` 실행


---
## 5.  활용 클라이언트 

총 2 종류의 클라이언트를 사용한다.(둘다 타입은 같음)
1. pub/sub 클라이언트들 : 실시간 통신을 위해 
2. 일반 클라이언트 : 일반 캐시/스토리지 용

---
### 5.1.  RedisService 클라이언트 

```JS
@Injectable()
export class RedisService implements OnModuleInit, OnModuleDestroy {

  private readonly logger = new Logger(RedisService.name);
  private readonly client: RedisClientType;

  constructor() {
    this.client = createClient({
      url: process.env.REDIS_URL ?? 'redis://localhost:6379',
    });

    this.client.on('error', (error) => {
      this.logger.error(`Redis error: ${error}`);
    });
  }
```
- 여기서 생성한 Client는 일반적인 Redis(캐시) 명령용 클라이언트 

---
### 5.2.  RedisAdapter용 pub/sub 클라이언트 
```JS
export class RedisIoAdapter extends IoAdapter {

  private adapterConstructor?: ReturnType<typeof createAdapter>;
  
  async connectToRedis(): Promise<void> {
    const pubClient = createClient({ url: process.env.REDIS_URL });
    const subClient = pubClient.duplicate();
    await Promise.all([pubClient.connect(), subClient.connect()]);
    this.adapterConstructor = createAdapter(pubClient, subClient);
  }
```
- Socket.IO용 Redis 어뎁터 전용 Client들이 2개 있다
	1. pubClient : 메시지 publish 용 
	2. subClient : subscribe용 
	
---
### 5.3.  한번에 관리하기 
```js
@Injectable()

export class RedisService implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(RedisService.name);
  private client: RedisClientType;
  private pubClient: RedisClientType;
  private subClient: RedisClientType;

  async onModuleInit(): Promise<void> {
    this.client = createClient({ url: process.env.REDIS_URL });
    this.pubClient = createClient({
      url: process.env.REDIS_URL ?? 'redis://localhost:6379',
    });
    this.subClient = this.pubClient.duplicate();
    await Promise.all([this.client.connect(), this.subClient.connect()]);
  
  ....
  getClient(): RedisClientType {
    return this.client;
  }

  getPubClient(): RedisClientType {
    return this.pubClient;
  }    
```
이렇게 client생성과 connect는 RedisService에서 처리하고 adapter는 받아오기만 하면 된다.

```js
export class RedisIoAdapter extends IoAdapter {

  private adapterConstructor?: ReturnType<typeof createAdapter>;

  constructor(
    app: INestApplicationContext,
    private readonly redisService: RedisService,
  ) {
    super(app);
  }

  connectToRedis() {
    const pubClient = this.redisService.getPubClient();
    const subClient = this.redisService.getSubClient();
    this.adapterConstructor = createAdapter(pubClient, subClient);
  }
```

