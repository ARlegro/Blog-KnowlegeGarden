---
{"dg-publish":true,"permalink":"/프로젝트/나만무/실시간 편집/NestJS - socket 기본 및 테스트/","noteIcon":"","created":"2025-12-03T14:53:06.797+09:00","updated":"2025-12-10T13:57:46.438+09:00"}
---


설치
```bash
npm i --save @nestjs/websockets @nestjs/platform-socket.io socket.io
```

---
## 1.  Websocket vs Socket.io

---
### 1.1.  socket 이란 
> 서버와 클라이언트가 서로 데이터를 주고받기 위한 통신 연결 객체

- 클라이언트 한 명당 연결을 담당하는 Socket 인스턴스 하나가 생성됨
- 정보
	- `client.id`: 소켓 고유 ID (자동 생성됨)
	- `client.handshake`: 연결 시 보낸 요청 헤더, 쿼리스트링 등의 정보
	- `client.emit()`: 서버가 특정 클라이언트에게 메시지를 보낼 때 사용
	- `client.on()`: 클라이언트가 서버의 이벤트를 받을 때 사용

---
### 1.2.  socket의 emit, on

|구문|뜻|방향|비유|
|---|---|---|---|
|`socket.emit('이벤트명', 데이터)`|이벤트 **보내기**|**클라이언트 → 서버** (또는 서버 → 클라이언트)|“야 서버야, 이런 데이터 보낼게!”|
|`socket.on('이벤트명', (data) => {...})`|이벤트 **받기 / 구독**|**서버 ← 클라이언트** (또는 클라이언트 ← 서버)**|“그 이벤트 오면 이 콜백 실행할게.”|

- `emit()` : 이벤트를 발생시키는 쪽(보내는 역할)
- `on()`: 이벤트를 듣는 쪽 (받는 역할, 콜백 실행)

---
### 1.3.  Room
> 논리적으로 room으로 구분하여 불필요한 통신을 줄임 

- Socket은 여러 개의 Room에 동시에 들어갈 수 있음 
- 관련 메서드
	- `join` : 해당 소켓을 특정 room 방에 추가 
	- `to` : 특정 Room의 사람들에게만 보낼 수 있게 함 
	- `leave`: 소켓을 방에서 제거 

```js
@WebSocketGateway()
export class ChatGateway {
  @WebSocketServer()
  server: Server;

  @SubscribeMessage('join')
  handleJoin(@ConnectedSocket() socket: Socket, @MessageBody() roomId: string) {
    socket.join(roomId); // ✅ 해당 소켓이 roomId 방에 참여
    socket.emit('joined', `You joined room ${roomId}`);
  }

  @SubscribeMessage('message')
  handleMessage(@MessageBody() data: { roomId: string; msg: string }) {
    this.server.to(data.roomId).emit('newMessage', data.msg);
    // ✅ 해당 room에 속한 소켓들에게만 브로드캐스트
  }
}
```



---
## 2.  NestJS의 Socket.IO 게이트웨이

> 클라이언트 - 게이트웨이(서버)가 이벤트를 주고 받는 구조

- 연결 유지 상태에서 `emit`/`on`으로 양방향 실시간 통신을 함

```jsx
@WebSocketGateway(3002, {}) // 이 클래스를 gateway로만든다?
export class ChatGateway {

	@WebSocketServer()
  server: Server;

  @SubscribeMessage('newMessage')
  handleMessage(@MessageBody() message: any) {
    console.log(message);
		
		// 첫 인자 : 이벤트 이름, 두 번째 인자 : 전송한 데이터
    this.server.emit('message', { text: '모두에게 전송' });
    // message : 서버가 클라이언트에게 보낼 이벤트 이름 
  }
}
```

1. `@WebSocketGateway`
    - *역할 - 게이트웨이 정의 및 서버 초기화*
	    - 클래스를 웹소켓 게이트웨이 서버로 등록 
	    - NestJS는 이거를 보고 내부적으로 `Socket.IO` 서버를 띄어서 클라 연결을 받을 준비
    - 첫 번째 인수 : 3002 포트에서 대기
    - 두 번째 인수 (`{}`)
        - 서버 옵션 넣는 자리
        - ex. CORS, path, namespace 등
          
2. `@WebSocketServer`
	- 실제 Socket.IO 서버 인스턴스를 주입받기 위한 것 
	- 이 애노테이션이 선언된 변수는 `Socket.IO Server` 인스턴스가 들어옴 
          
3. `SubscribeMessage('newMessage')`
    - 클라이언트가 `newMessage`이벤트로 보낸 데이터를 받는 핸들러
    - 이벤트 이름 : `newMessage`
    
    ```jsx
    // 브라우저/프론트 (socket.io-client)
    const socket = io('<http://localhost:3002>');
    socket.emit('newMessage', { text: '안녕', roomId: 'lobby' });
    ```
    
4. `@MessageBody()`
    - 들어오는 이벤트 Payload(메시지 본문)를 자동으로 꺼내줌
    - DTO로 쓰면 검증도 가능

```js
// 서버가 보낸 'message' 이벤트를 수신
socket.on('message', (data) => {
  console.log('서버에서 받은 메시지:', data.text);
});
```


---
### 2.1.  Socket.IO 활성화

단순히 `@WebSocketGateway`로 등록한다고 되는게 아니다 module에서 provider로 등록해야 함

```jsx
@Module({
  providers: [ChatGateway],
})
export class ChatModule {}

```


---
## 3.  실제 

---
### 3.1.  기본 - 단일 전송 
```js
  @SubscribeMessage('newMessage')
  handleNewMessage(client: Socket, message: any) {
    console.log(message)
  
  
		
		client.broadcast.emit('reply', '응답입니다'); // 나 제외 브로드 캐스트
		// client.emit('reply', '응답입니다'); // 단일 전송(나 포함)
  }
```

![Pasted image 20251103113003.png](/img/user/supporter/image/Pasted%20image%2020251103113003.png)
![Pasted image 20251103113011.png](/img/user/supporter/image/Pasted%20image%2020251103113011.png)


---
### 3.2.  브로드 캐스트 
> 아주 간단. 서버 인스턴스를 활용하면 됨 

```js
  @SubscribeMessage('newMessage')
  handleNewMessage(client: Socket, message: any) {
    console.log(message);
    //client.emit('reply', '응답입니다'); // 단일 전송
    this.server.emit('reply', '브로드캐스트 응답입니다'); // 단일 전송
  }
```


---
### 3.3.  연결될 때, 

단순히 Message 수신은 `@SubscribeMessage`를 사용하면 된다.
하지만 Socket.IO는 추가적으로 연결 관리 기능을 제공함
- 연결될 때 → `handleConnection(client)`
- 메시지 수신 시 → `@SubscribeMessage('event')`
- 연결 끊길 때 → `handleDisconnect(client)`

```JS
@WebSocketGateway(3002)
export class ChatGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  server: Server;

  handleConnection(client: Socket, ...args: any[]) {
    console.log('new user connected', client.id);
    client.broadcast.emit('reply', {
      message: `새로운 유저 ${client.id} 가 들어왔습니다.`,
    });
  }

  handleDisconnect(client: Socket) {
    console.log('disconnected:', client.id);
  }

  @SubscribeMessage('newMessage')
  handleMessage(@MessageBody() data: any) {
    // 전체 클라이언트에게 브로드캐스트
    this.server.emit('message', data);
  }
}
```

---
### 3.4.  room으로 나누기 

✅Join 
```js
  // @ConnectedSocket() : 현재 연결된 클라이언트의 소켓 객체를 주입받음 => Socket
  // @MessageBody() : 클라이언트에서 보낸 데이터를 가져옴
  @SubscribeMessage('join')
  async join(@ConnectedSocket() socket: Socket, @MessageBody() body: string) {
    // socket.join : 특정 클라이언트를 room에 넣는 함수
    const dto: SendDto = JSON.parse(body) as SendDto;
    await socket.join(dto.roomId);
    socket.emit('joined', dto.roomId);
  }
```

✅보내기 
```js
  @WebSocketServer()
  server: Server;
  
  @SubscribeMessage('chat/send')
  send(@ConnectedSocket() socket: Socket, @MessageBody() body: string) {
    const dto: SendDto = JSON.parse(body) as SendDto;
    this.server.to(dto.roomId).emit('chat/receive', {
      message: dto.message,
      from: socket.id,
      at: Date.now(),
    });
```[