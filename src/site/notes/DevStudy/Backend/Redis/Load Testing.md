---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Redis/Load Testing/","noteIcon":"","created":"2025-12-03T14:52:50.292+09:00","updated":"2025-12-13T10:22:57.722+09:00"}
---



> 부하 테스트

>사용 기술 : k6

설치법 
https://grafana.com/docs/k6/latest/set-up/install-k6/

## 1.  K6 부하 테스트 - Redis 도입 버전 

### 1.1.  API에 부하를 주기 위해 k6 스크립트 작성
**script.js**
```javascript
import http from 'k6/http';
import { sleep } from 'k6';

export default function () {
  http.get('http://{EC2 IP 주소}:8080/boards');
}
```

### 1.2.  부하 실행 
```bash
k6 run --vus 30 --duration 10s script.js 
```
- **vus 30 : virtual users**
	- 가상 유저 30명을 만들 것 
	- 30명의 유저가 동시에 요청하는 것처럼 세팅
	  
- **duration** : 얼마나 부하를 지속할지 

>이렇게 하면 script에 적혀있는 주소로 계속 요청 보냄 

실행하면 아래와 같이 진행중 
![Pasted image 20250604232257.png](/img/user/supporter/image/Pasted%20image%2020250604232257.png)

### 1.3.  결과 
```bash
  █ TOTAL RESULTS 

WHTTP
http_req_duration.........................: avg=4.17ms min=255.25µs med=3.18ms max=3.09s p(90)=6.7ms  p(95)=8.57ms
	{ expected_response:true }..............: avg=4.17ms min=255.25µs med=3.18ms max=3.09s p(90)=6.7ms  p(95)=8.57ms
http_req_failed...........................: 0.00%  0 out of 188989
http_reqs.................................: 188989 9439.207906/s

EXECUTION
iteration_duration........................: avg=4.22ms min=286.7µs  med=3.22ms max=3.1s  p(90)=6.76ms p(95)=8.63ms
iterations................................: 188989 9439.207906/s
vus.......................................: 40     min=40          max=40
vus_max...................................: 40     min=40          max=40

NETWORK
data_received.............................: 211 MB 11 MB/s
data_sent.................................: 14 MB  717 kB/s
```

Duration이 끝나면 이렇게 결과가 나온다.
> 중요하게 봐야하는 것은 http_reqs 결과이다.

**http_reqs ---- : 188989 9439.207906/s**
- 188989 : Duration동안 요청을 처리한 개수 
- 9439.20/s : **1초당 9439.20~개의 요청을 처리**했다는 뜻 ⭐⭐
	>이걸 Throughput이라고 함. 
	- 말할 때, 'Throughput은 9439.20입니다!' 라고 하면 됨 
	- or 9439.20 TPS 입니다 라고 하면 됨 


## 2.  K6 부하 - Redis ❌

조회 로직의 @Cacheable을 주석처리했다.
```java
//Cacheable(cacheNames = "boards", key = "'board:page:'+ #page + ':size:' + #size", cacheManager = "boardCacheManager")  
public List<Board> getBoards(int page, int size) {  
    Pageable pageRequest = PageRequest.of(page - 1, size);  
    Page<Board> pageOfBoard = boardRepository.findAllByOrderByCreatedAtDesc(pageRequest);  
    return pageOfBoard.getContent();  
}
```

실행방법과 로직은 이전과 같고 결과만 보자 

### 2.1.  결과 
```bash
█ TOTAL RESULTS 

HTTP
http_req_duration....................: avg=1.8s min=412.31ms med=1.81s max=3.27s p(90)=2.07s p(95)=2.11s
	{ expected_response:true }.........: avg=1.8s min=412.31ms med=1.81s max=3.27s p(90)=2.07s p(95)=2.11s
http_req_failed......................: 0.00%  0 out of 466
http_reqs............................: 466    21.354778/s

EXECUTION
iteration_duration...................: avg=1.8s min=416.27ms med=1.81s max=3.26s p(90)=2.07s p(95)=2.11s
iterations...........................: 466    21.354778/s
vus..................................: 20     min=20       max=40
vus_max..............................: 40     min=40       max=40

NETWORK
data_received........................: 519 kB 24 kB/s
data_sent............................: 35 kB  1.6 kB/s
```
**http_reqs 를 보면** 
- **총 처리 수** : 466  (vs Redis 도입 :  188989)  ✅405배 차이 
- **TPS** : 21.35 (vs Redis 도입 : 9439.20) ✅442배 차이 

>비교 엄청된다. 대용량 처리 시 조회 성능 큰 차이 


