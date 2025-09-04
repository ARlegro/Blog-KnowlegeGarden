---
{"dg-publish":true,"permalink":"/Computer_Science/Network/C-Network/Concurrent Proxy 구현/","noteIcon":"","created":"2025-09-05T02:07:52.760+09:00","updated":"2025-09-05T02:10:08.060+09:00"}
---

참고
- [[Computer_Science/Network/C-Network/기본 Proxy 구현해보기\|기본 Proxy 구현해보기]]
- 깃허브 링크 : https://github.com/ARlegro/web-server-impl 


기존 기본 Proxy 코드([[Computer_Science/Network/C-Network/기본 Proxy 구현해보기\|기본 Proxy 구현해보기]])를 동시성 프로그래밍으로 만드는 과정

> 멀티 스레드 기반이고, 변경할게 많이 없다.

### fd 할당 - malloc으로 변경

```c
	int *connfdp;
	
	while (1) {
			...
			connfdp = Malloc(sizeof(int));
			*connfdp = Accept(listenfd, ~~~~)
			Pthread_create(&tid, NULL, thread, connfdp);
	}
```
흐름 간단 설명 
- main 쓰레드가 `while`문을 돌면서 계속 새로운 클라이언트의 연결을 받는다.
- 이때 `Accept()`가 반환하는 fd를 각 쓰레드에게 반환해야 함 

>[!tip] Malloc 사용 이유 - 경쟁 조건 해결
>- 💢문제 : 스택에 저장된 지역 변수 사용 시 connfdp를 여러 쓰레드가 경쟁할 수 있다.
>- ✅힙에 메모리를 할당하면 ➡ **새로운 연결마다 독립적인 메모리 공간**


### thread 루틴 정하기 

1. *전달받은 connfd 포인터 값을 복사하여 정수값 만듬*
	- `int connfd = *((int *)vargp);
	- *형변환 이유*
		- thread생성 시에 실행되는 메서드는 `void *`으로 파라미턴가 넘어오므로 
	- *값 복사 하는 이유* 
		- 인자로 넘어온 포인터는 메인 쓰레드가 갖고 잇는 포인터인데, 만약 다음 요청이 들어오면 메인 쓰레드는 그 포인터 값을 바꿀 것이다.
		- **이 상황에서 처음 자식 쓰레드가 작업을 하면 다른 쓰레드에도 영향을 끼치게 됨**
	  
2. *전달받은 connfd 포인터 값 Free*
	- `connfdp`는 `Malloc`에 의해서 동적으로 할당된 것이다. 
	- 따라서, 일단 값 복사했으면 **메모리 누수를 막기 위해 connfd를 해제**하면 된다
	  
3. *메인 작업* 
4. *fd 자원 처리 (connfd)*

