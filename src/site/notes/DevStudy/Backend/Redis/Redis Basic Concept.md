---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Redis/Redis Basic Concept/","noteIcon":"","created":"2025-12-03T14:52:50.251+09:00","updated":"2025-12-13T10:23:25.080+09:00"}
---



### 0.1.  Redis란?
>Redis는 데이터 처리 속도가 엄청 빠른 NoSQL 데이터베이스

“키-값” 구조의 비정형 데이터를 저장하고 관리하기 위한 오픈 소스 기반의 비관계형 데이터베이스 관리 시스템(DBMS)

**인메모리(in-memory)에 모든 데이터를 저장 ➡ 성능이 매우 빠르다.**
- DB는 대부분 Disk에 데이터를 저장한다. 따라서 비용이 많이 드는 I/O작업으로 인해 느려진다.
- **데이터 처리 속도 :  메모리 > Disk**

**추가로, 싱글 스레드로 작동하여 경합이 발생하지 않는다.**



### 0.2.  주요 사용 사례 
> 흔히 Redis는 캐싱만 가능할 줄 아는데 아니다.

- 캐싱 (Caching)
- 세션 관리 (Session Management)
- 실시간 분석 및 통계 (Real-time Analystics)
- 메시지 큐 (Message Queue)
- 지리공간 인덱싱 (Geospatial Indexing)
- 속도 제한 (Rate Limiting)
- 실시간 채팅 및 메시징 (Real-time Chat And Messaging)


### 0.3.  Redis 우아하게(참고)

https://www.youtube.com/watch?v=tVZ15cCRAyE

1. 데이터 타입에 따른 적절한 자료구조 사용 
		![Pasted image 20250604222318.png](/img/user/supporter/image/Pasted%20image%2020250604222318.png)
	- 최근 검색 목록을 표시하는 기능이 있다고 할 때, "**정렬 + 데이터 중복제거**" 가 필요하다
	- 이걸 가중치를 이용해서 어떻게 할 수 있다는데 심화내용이라 pass
	  
2. **O(n) 명령어 주의**
	- Redis의 명령어를 돌이켜보면, `flushall, keys, delect collections, get all collections` 등과 같은 O(N)의 시간복잡도를 가진 명령어들이 있었다
	- Redis는 '싱글 스레드'이기 때문에 해당 명령어를 처리할 때까지 다른 명령어들이 대기 상태에 빠짐
	- 그로 인해, **성능이 오히려 낮아지는 문제 발생** 
	  
3. 메모리 관리 - 메모리 단편화 발생
		![Pasted image 20250604222924.png](/img/user/supporter/image/Pasted%20image%2020250604222924.png)
	- **메모리 단편화** : 메모리가 작은 공간으로 나눠져서 관리되어 **사용가능한 공간은 충분함에도 해당 메모리를 할당하지 못하는 것** 
	- 어떻게 관리하는가?
		- RSS(Resident Set Size, 실제 물리 메모리 사용량) 모니터링 必








