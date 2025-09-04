---
{"dg-publish":true,"permalink":"/Computer_Science/Network/C-Network/Proxylab 요구사항 정리/","noteIcon":"","created":"2025-09-01T20:45:37.398+09:00","updated":"2025-09-05T02:22:53.799+09:00"}
---



https://wentao.site/proxylab/#:~:text=Source%3A-,proxylab.pdf,-Source%3A%20proxylab



## 4.1 HTTP/1.0 GET 요청 
`http://www.cmu.edu/hub/index.html`과 같은 URL을 입력하면, 브라우저는 다음과 유사한 줄로 시작하는 HTTP 요청을 프록시로 보냅니다. `GET http://www.cmu.edu/hub/index.html HTTP/1.1`

이 요청을 받은 프록시는 최소한의 필드만 요청 
ex. `GET /hub/index.html HTTP/1.0`   <<< **1.0으로 만들기** 


> [!WARNING]
HTTP 요청의 모든 줄은 캐리지 리턴('\r')과 줄 바꿈('\n')으로 끝나며, 모든 HTTP 요청은 **빈 줄 `"\r\n"`로 종료**된다는 점

---
## 4.2 요청 헤더 

중요한 요청 헤더
- **Host**
- **User-Agent**
- **Connection**
- **Proxy-Connection**

---
#### 1. host
> Host 헤더는 최종 서버의 호스트 이름을 설명 : 항상 Host 헤더를 보내야

ex. `http://www.cmu.edu/hub/index.html`에 접근하기 위해 프록시는 다음 헤더를 보냅니다. `Host: www.cmu.edu`

> [!WARNING] 단, Web Brower가 자체 Host 헤더를 TTP 요청에 첨부하는 경우, 프록시는 **브라우저와 동일한 Host 헤더를 사용**해야 함 

---
#### 2. User-Agent 
> 클라이언트(운영 체제, 브라우저 등)를 식별하며, 웹 서버는 이 정보를 사용하여 제공하는 콘텐츠를 조작하는 헤더 

---
#### 3. Connection - close 
> 항상 Close

---
#### 4. Proxy-Connection - close
> - 첫 요청/응답 교환이 완료된 후 연결 유지 여부를 지정 
> - 즉, 요청마다 새로운 연결 여는 것이 허용 

---
#### 5. 그 외 
> 다른 요청 헤더들은 변경 없이 그대로 전달해야 한다.


---
## 4.3 포트 번호 

과제에는 2가지 주요 포트 번호가 있다
1. HTTP 요청 포트
2. 프록시의 수신 포트 

---
### 1. HTTP 요청 포트 
- HTTP 요청 URL의 선택적 필드입니다. 
- 예를 들어, URL이 `http://www.cmu.edu:8080/hub/index.html`과 같은 형식일 수 있습니다. 
- 이 경우, 프록시는 기본 HTTP 포트인 80 대신 **8080 포트의 호스트 `www.cmu.edu`에 연결**해야 합니다. 
- 포트 번호가 URL에 포함되든 그렇지 않든 프록시는 올바르게 작동해야 합니다.

---
### 2. 프록시의 수신 포트 
프록시가 들어오는 연결을 수신하는 포트

- 프록시는 명령줄 인수로 수신 포트 번호를 받아들여야 합니다. 
- 예를 들어, 다음 명령을 사용하면 프록시는 **15213 포트에서 연결을 수신**해야 합니다. `linux> ./proxy 15213` 
- 다른 프로세스에 의해 사용되지 않는 **비특권 수신 포트(1,024보다 크고 65,536보다 작은)** 를 선택할 수 있습니다. 
- 각 Proxy는 고유한 수신 포트를 사용해야 하고 여러 사람이 동시에 작업하기 때문에, 사용자 ID에 따라 개인 포트 번호를 선택하는 데 도움이 되는 `port-for-user.pl` 스크립트가 제공됩니다. 
- `linux> ./port-for-user.pl droh` `droh: 45806` `port-for-user.pl`이 반환하는 포트 `p`는 **항상 짝수**입니다. 
- 따라서 Tiny 서버와 같은 추가 포트 번호가 필요한 경우, **포트 `p`와 `p + 1`을 안전하게 사용**할 수 있습니다. **무작위 포트를 선택하지 마세요. 그렇게 하면 다른 사용자와 충돌할 위험이 있습니다.** ⭐

---
## 5.2 동시 요청 처리 
> 이제 여러 요청을 동시에 처리할 수 있도록 수정


---
# 캐싱 

---
### 조건 1. 크기 
```c
#define MAX_CACHE_SIZE 1049000
#define MAX_OBJECT_SIZE 102400
```

1. **캐시 최대 크기** = 1MB  
2. **최대 캐싱 객체 크기** = 100KB

>[!tip] pdf 曰 캐시 구현 hint
>1. *연결마다 전용 버퍼를 할당한다*
>2. *데이터 크기 검사 및 누적*
>	- 서버가 데이터 보내주면 **크기 검사 및 임시 버퍼에 데이터 누적** 
>	- 누적 후 buffer_object가 100KB가 안넘는지 확인 
>3. *buf 폐기 및 갱신*
>	- buf_object 크기가 100KB **초과 시 임시 버퍼를 즉시 폐기** 🥊
>	- *언제 메인 캐시에 올리는가❓*
>		- 임시 버퍼에 누적된 데이터들이 크기 검사 통과하면서 모두 도착했을 때!!!<br>
>4. ✅이 전략 시 Proxy가 사용하는 최대 메모리 양 = `MAX_CACHE_SIZE + T * MAX_OBJECT_SIZE`
>	- `MAX_CACHE_SIZE` : 프록시의 메인 캐시가 가질 수 있는 최대 크기
>	- `T` : 동시에 처리할 수 있는 최대 활성 연결 수 
>	- `T * MAX_OBJECT_SIZE` : 모든 임시 버퍼 크기의 합 (❗이는 **캐시에 포함되지 않는 추가 메모리**)


흠... 캐시 버퍼는 스레드 함수 내에서 구조체 선언하고, 전체 캐시는 전역 변수로하고 **읽기는 공유되고 쓰기는 락걸기**

---
### 조건 2. 제거 정책 

> LRU 알고리즘 사용하기 (Least Recently Used Algorithm)
> - 가장 오랫동안 참조되지 않은 페이지 교체하는 기법 

> [!WARNING] 주의 : 객체를 읽는 것과 쓰는 것 모두 객체 사용으로 간주

---
### 조건3. 동기화 
1. **여러 thread가 동시에 캐시를 읽을 수 있어야** 함  ➡ `pthread_rwlock_t` 必
2. 쓰기 작업은  thread-safe하게 하기 

> [!WARNING] 배타적 락 불가능 
> 할거면
> 1. 캐시 분할하거나
> 2. Pthreads의 readers-writers locks를 사용하거나 
> 3. 세마포어를 사용하여 readers-writers 구현 

