---
{"dg-publish":true,"permalink":"/Computer_Science/Network/C-Network/기본 Proxy 구현해보기/","noteIcon":"","created":"2025-12-03T14:52:46.128+09:00","updated":"2025-12-13T18:25:26.834+09:00"}
---



참고 
- [[Computer_Science/Network/개념/Proxy 서버 개념\|Proxy 서버 개념]]
- [[Computer_Science/Network/C-Network/Proxylab 요구사항 정리\|Proxylab 요구사항 정리]]
- [[Computer_Science/Network/C-Network/Tiny 웹 서버 구현\|Tiny 웹 서버 구현]] : Proxy의 end 서버로 이전에 구현했던 Tiny 서버를 활용할 것 
- 깃 허브 : https://github.com/ARlegro/web-server-impl -

클라와 서버 사이에서 연결을 도와줘야하는데...

## Proxy서버가 할 일

### 헤더 처리 

#### 1. 요청 라인 정규화 (absolute -> origin)

- "Client -> Proxy"는 절대 경로 스타일일 수 잇다. ex. - `GET http://host/path HTTP/1.1`
- Proxy → origin 서버: **origin-form**으로 바꿔 보냄 (`GET /path HTTP/1.1`)

>[!QUESTION] 왜 origin-server에 origin-form으로 보내야할까??
>- 클라이언트가 Proxy 서버를 대상으로 할 때는 최종 목적지가 아니므로 absolute form으로 보낸다.
>- But, origin-server에 보낼 때는 origin-form으로 보내야 함
>- 따라서, 정규화 必


#### 2. Origin으로 보낼 헤더 선별/재작성



*💚유지 해야 할 것*
1. `host[:포트]`
	- 요청라인에서 absolute-form으로 온 uri를 origin-form으로 바꾼다.
	- (옵션) port번호가 있으면 그것도 포함 

	  
>[!tip] 호스트 vs 도메인 
>1. *호스트(host)*
>	- 네트워크에 **연결된 특정 컴퓨터나 장치**
>	- IP 주소로 식별된다. But 사람이 못 읽으니 그냥 DNS에 이름 붙여줌
>	- ex. `www.google.com`
>	  
>2. *도메인*
>	- 사람들이 **쉽게 기억할 수 있도록 만들어진 이름** 
>	- ex. `google.com`
	  
2. `Connection: close`
	- HTTP 1.1의 표준 header
	- 기본은 keep aliver로 connection을 유지하는 것인데, **`close`시 연결을 재사용하지 않게 됨**
		- 효과 : origin 서버가 응답을 보내면 소켓 닫음 
		  
> [!INFO] Connection의 다른 옵션 : Keep-Alive
> - *의미* : **끊으라는 요청이 없으면 TCP 연결을 유지**한다는 뜻
> - HTTP/1.1 부터 기본값 
> - *필요해진 이유*
> 	- 여러 요청/응답에 TCP연결을 여러 번 할 필요가 없다.
> 	- 이렇게 되면 오버헤드가 줄어들음(이게 HTTP/1.1의 장점)
> - *단점 및 한계*
> 	1. **자원 점유**
> 		- TCP를 계속 열어 놓으면 자원을 계속 차지하게 되어 **커넥션 풀이 마르는 문제**가 발생할 수 있음
> 		  
> 	2. **실시간성이 아님**
> 		- 단순히 연결만 유지하는 것일뿐, 실시간 통신이 아니다
> 		- 실시간 통신 시 websoket ㄱㄱ
> 		  
> 	3. **병렬성 부족**
> 		- 하나의 Keep-Alive 연결은 한 번에 하나의 요청/응답만 처리가 가능하다.
> 		- 만약, 이미지/JS/CSS 리소스를 **병렬로 요청 시 여러 TCP연결을 했어야 했음**
> 		- 최근에는 이 문제가 HTTP/2의  Multiplexing 방식으로 해결됨
> 		  
> 	4. **최적화 한계**
> 		- HTTP2/3와 달리 압축, 헤더 최적화, 푸시 등이 안 일어남 
		  
3. `Proxy-Connection: close`
	- RFC에는 없는 비표준 확장 header
	- Client와 Proxy 사이의 연결을 close로 처리 
	- 현대 브라우저/proxy에는 거의 안씀 (교육용)
	  
4. `User-Agent`
	- 과제에서 제공된 user_agent 는 무조건 보내야 함 
	  
5. (옵션) `via`
	- 어떤 경로를 통했는지 명시해주는 header
	- ex. `Via: 1.0 localhost.localdomain:8765 (~~~~)`
	- 프록시를 쓰고 있다는 것을 알려주는 거라 일부 Proxy에서는 포함되지 않을 수있다.
	  
6. (옵션) `X-Forwarded-For`
	- Client의 IP를 보내줌 
	- 일부 익명성을 보장하는 Proxy에서는 쓰지 않는다.
	  
7. (옵션) `X-BlueCoat-Via`
	- Client의 IP는 익명이지만 **프록시를 사용중이라는 것을 표시**할 때 사용



### 동적 컨텐츠 처리 시 파라미터 동적 처리 



```c
n1 = atoi(arg1)
/* Convert a string to an integer.  */
extern int atoi (const char *__nptr)
```

