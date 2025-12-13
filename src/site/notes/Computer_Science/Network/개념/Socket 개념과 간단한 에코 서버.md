---
{"dg-publish":true,"permalink":"/Computer_Science/Network/개념/Socket 개념과 간단한 에코 서버/","noteIcon":"","created":"2025-12-03T14:52:46.199+09:00","updated":"2025-12-13T09:26:25.452+09:00"}
---


### 소켓이란 
> OS가 제공하는 통신 인터페이스 

- *정의 *
	- 통신용 엔드포인트(End-Point)
	- 프로토콜 요소(ex. TCP, UDP)에 대한 **추상화 인터페이스**
	- 사실 장치 "**파일**"의 일종 ⭐
- *통신 방식에 따른 **종류***
	1. *TCP 소켓* : 연결지향적 소켓. 신뢰성 있는 데이터 전송 시 사용(ex. HTTP, SMTP, FTP)
	2. *UDP 소켓* : 비연결 지향 소켓. 속도를 우선시 


>[!tip] 소켓 프로그래밍에서 Port번호는 Socket 식별자이며 프로세스 식별자이다.
>소켓 하나에 Port하나가 binding되는데, 그 소켓을 열은 주체는 프로세스이다.
>따라서, Port번호 = 프로세스 식별자

가상 메모리 매핑으로도 접근 가능하고 직접 파일 접근도 가능 

---
## 소켓 생성 

소켓(파일) 구조체 반환 - 소켓 핸들 

---
### TCP/IP 소켓 생성 예시 
```C
#include <sys/socket.h> // socket() 제공 
socket = socket(AF_INET, SOCK_STREAM, 0)

/* Create a new socket of type TYPE in domain DOMAIN, using
   protocol PROTOCOL.  If PROTOCOL is zero, one is chosen automatically.
   Returns a file descriptor for the new socket, or -1 for errors.  */
extern int socket (int __domain, int __type, int __protocol) __THROW;
```
*파라미터 분석*
1. `__ domain`
	- 이 소켓이 어떤 주소 방식을 사용할지 지정하는 것 
	- *예시*
		- `AF_INET` : IPv4 주소 체계 사용
		- `AF_INET6` : IPv6 주소 체계 사용
	  
2. `__type`
	- **소켓 타입**을 의미 
	- *예시*
		- `SOCK_STREAM` : **연결 지향형 STREAM 소켓 (TCP 기반 소켓)**. 바이트 스트림으로 동작하여 연속된 데이터 흐름
		- `SOCK_DGRAM` : 비연결형 데이터 그램 소켓 (UDP 기반 소켓)
	  
3. `__protocol`
	- 사용할 프로토콜 지정(번호로)
	- 만약 0을 넣으면 커널이 적절한 프로토콜을 자동으로 선택해줌
		- `__domain`, `__type` 지정 시 거의 그대로 
		- 거의 0을 넣는다 (앞에 2개 파람은 지정)


*Return* : 파일 디스크립터 (참고 : [[파일 디스크립터\|파일 디스크립터]])

---
### 소켓 주소 구조체 

>[!QUESTION] 왜 주소 구조체가 필요한가?
>- 소켓이 통신을 하려면 **주소, 포트 정보가 필요**하다 ✅
>- 실제로 소켓의 `connect()`, `bind()`, `accept()`함수는 파라미터에 소켓 주소 구조체를 가리키는 포인터가 있다.
>- 이 소켓 API를 만들 당시 `void *`가 C에서 없어서 이렇게 복잡하게 만든 것 

C 소켓 API에서는 여러 소켓 주소 구조체를 제공해준다.

---
#### 일반형 구조체 - sockaddr 
#generic 

> 모든 주소 구조체를 묶어서 표현할 수 있는 최소 구조체 


```c
struct sockaddr {
    uint16_t sa_family;  /* 주소 체계(AF_INET, AF_INET6, AF_UNIX...) */
    char     sa_data[14];/* 실제 주소 데이터 (IP, 포트 등) */
};
```

---
#### IP 전용 구조체 - sockaddr_in
> 네트워크 통신에서 **IPv4 주소와 포트 번호를 저장**하는 데 사용되는 구조체

#IP전용 


```c 
// in.h 라이브러리에 구현되어있는 구조체 
/* Structure describing an Internet socket address.  */
struct sockaddr_in
  {
    __SOCKADDR_COMMON (sin_);
    in_port_t sin_port;     /* 포트 번호 */
    struct in_addr sin_addr;    /* IP 주소 구조체 */
  
    /* Pad to size of `struct sockaddr'.  */
    // 패딩용 속성
    unsigned char sin_zero[sizeof (struct sockaddr)
         - __SOCKADDR_COMMON_SIZE
         - sizeof (in_port_t)
         - sizeof (struct in_addr)];
  };
```

```C
// ✅사용 
#include <sys/socket.h>
#include <arpa/inet.h>

int main(){
  struct sockaddr_in serv_addr;

	// 주소 체계 설정 
  serv_addr.sin_family = AF_INET; // IPv4
  //serv_addr.sin_family = AF_INET6; // IPv6

  // IP 주소 설정
  serv_addr.sin_addr.s_addr = inet_addr("127.0.0.1");

  // 포트 번호 설정 : 8080
  serv_addr.sin_port = htons(8080); 
```

> 구조체를 **만들 때는 주로 구체형 구조체를 선호**한다.(호출 시에만 일반형으로 캐스팅)


*Note : htons 필요 이유*
- socket라이브러리를 만든 회사들이 big엔디안을 사용했는데 network는 little엔디안을 사용
- 따라서 이러한 차이를 극복하고 **호환이 필요**해서 사용 



---
## 간단한 TCP 에코 서비스로 소켓 이해하기 

*TCP 에코 서비스*: 클라이언트가 보낸 데이터를 서버가 그대로 다시 클라이언트에게 되돌려 보내는(echoing) 서비스 - 기본적인 소켓 구성을 이해하기 좋은 모델 

---
### 동작 과정 
![Pasted image 20250829203531.png](/img/user/supporter/image/Pasted%20image%2020250829203531.png)
---
#### 서버 역할 
> 클라이언트의 요청을 받아들이고 처리 

1. *소켓 생성 (`socket()`)*
	- 네트워크 통신을 위한 Endpoint인 소켓을 생성 
	- 통신을 위한 준비 단계 
	  
2. *주소 할당 (`bind()`)*
	- 생성된 소켓에 특정 IP주소와 Port번호를 연결 
		- 서버가 사용할 IP주소와
		- 클라의 연결 요청을 받을 Port번호
	- Note : 하나의 Port를 여러 프로세스가 동시에 공유할 수 없다
	  
3. *연결 요청 대기*(`listen()`)
	- `bind`된 소켓을 **수신 소켓으로 전환**하고, 클라이언트의 **연결 요청을 기다리는 상태로** 만듬 (OS에 알림)
	  
4. *연결 수락(`accept()`)*
	- `listen()` 상태에 있는 소켓으로 들어온 **클라이언트의 연결 요청을 수락**하고, 
	- 클라이언트와 **실제 데이터 통신에 사용할 *새로운 소켓***을 생성✅
	  
		```c
		extern int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
		```
		- `sockfd` : 서버 소켓 fd(리스닝 소켓 )
		- `*addr` : 클라이언트의 주소 정보를 담는 곳
		- `*addrlen` : `*addr` 버퍼 크기를 담은 변수의 포인터
			- 포인터인 이유는 호출당하는 쪽에서 addrlen이하 만큼 값을 채운 뒤 **얼마나 채웠는지 호출하는 쪽에 알려주기 위해** 
		  
	- *❓기존 Listen 소켓은❓*
		- 계속해서 **다른 클라이언트의 연결 요청을 받아줄 대기**를 함 
		- 이로 인해, 다수 클라와 통신이 가능한 것 
		  
5. *데이터 수신/재전송(`recv()`/`send()`)*
	- `recv()` : 클라가 보낸 데이터를 받는다
		- 에코 모델에서 `while`문에 있는 `recv()`는 호출 후 클라이언트가 데이터를 보낼때까지 대기한다.(만약 클라가 `close`시 0 반환되어 while문 종료)
	- `recv()` : 받은 데이터를 다시 되돌리기 ❗(Echo시스템 만의)
   
6. *연결 종료 및 소켓 닫기(`shutdown()`/`closesocket()`*)
	- 서버가 연결을 종료하면 클라도 `shutdown()`을 호출하여 종료한다.
	- 연결 종료 후 통신 소켓을 닫는다.
	  
7. *Listen 소켓 닫기(`closesocket()`)*
	- 6번에서 닫은 소켓은 통신 소켓이다.
	- 여전히 대기중인 소켓이 있기 때문에 이를 닫을 필요가 있다.


> [!WARNING] 소켓은 하나만 사용하지 않는다.
> 4번을 보면 알 수 있듯이, 요청을 대기하는 서버 소켓 외에도 서버 통신 소켓이 있다.
> 이 외에도 클라이언트의 소켓도 있음

---
#### 클라이언트 역할 

> 접속을 시도하는 주체 

거의 비슷함 사진 보면서 ㄱ 

1. *소켓 생성(`socket()`)*
	- 서버와 통신할 수 있는 소켓을 생성 
	- 통신에 적합한 소켓을 만들면 됨(소켓 타입, 주소 체계 등)
	- 생성 후 서버와 연결하기 전에 Port, IP 등을 설정

>[!tip] 클라 소켓은 Port 미지정 시 OS가 알아서 지정해줌!
	  
2. *서버 연결(`connect()`)*
	- 생성된 소켓을 이용해 특정 **서버의 IP 주소와 포트 번호**로 연결을 요청 
	- *조건* : Server의 소켓이 `listen()`상태인 경우 연결 시도 가능 

> [!INFO] 이 때 일어나는 것이 3way handshake 
> - 서버-클라 각각 랜덤 sequence number를 부여받고 교환하면서 연결 상태 확인하는 과정 
> - 3way-handshake는 다른 곳에서 잘 적어놨으니 pass

3. *데이터 송신/수신(`send()/recv()`)*
	 - 연결된 소켓을 통해 서버로 데이터를 보내고 서버가 보낸 데이터를 받는다.
	   
4. *연결 종료 및 소켓 닫기 (`shutdown()`/`closesocket()`)*
	- 더 이상 통신할 필요가 없을 때 **연결을 종료하고 소켓을 닫는다**
	- 서버가 먼저 `shutdown()`하면 이 과정이 되기도 하고,
	- 클라가 먼저 `shutdown()`해도 이 과정이 되기도 함
	- 이를 통해 서버와의 통신이 공식 종료 


> [!INFO] 연결 종료 시 4-Way handshake 발생 
> ![Pasted image 20250830011209.png](/img/user/supporter/image/Pasted%20image%2020250830011209.png)

> [!WARNING] 소켓 설계 시 Client가 종료하도록 설계해야한다
> - 종료 주체가 누구냐에 따라 리소스 부담과 안정성이 크게 달라진다.
> - 일반적으로는 Client가 먼저 종료하게 하는 것이 좋은 설계 (서버는 패시브 전략) <BR>
> - *간단한 이유*
> 	- 손실 없는 **우아한 종료가 가능** 
> 	- TimeWait가 서버에 발생 시 다른 클라에도 영향을 미칠 수 있기에 Client가 TimeWait발생하도록 설계 必
> - 💢물론, 반드시 이렇게 설계해야 한다는 것은 아니다

