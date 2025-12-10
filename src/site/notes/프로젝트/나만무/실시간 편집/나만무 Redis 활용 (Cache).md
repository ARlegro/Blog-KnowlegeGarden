---
{"dg-publish":true,"permalink":"/프로젝트/나만무/실시간 편집/나만무 Redis 활용 (Cache)/","noteIcon":"","created":"2025-12-03T14:53:06.769+09:00","updated":"2025-12-10T10:51:24.124+09:00"}
---



## 1.  Redis Cache 사용 이유 

워크스페이스 협업 환경에서는 사용자가 장소를 **마킹/수정/삭제**하는 이벤트가 매우 빈번하게 발생한다. 이 모든 이벤트를 즉시 DB에 반영하면 아래와 같은 문제가 생긴다
1. DB 쓰기 폭주
2. 쿼리 레이턴시 증가 → 실시간 WebSocket 업데이트 지연
3. 워크스페이스 단위의 일시적 작업에도 영속 저장소를 계속 수정하는 비효율

따라서, Redis를 활용한 캐시를 통해 DB 부하를 줄이고 쌓인 정보들을 한 번에 DB Batch 작업을 하기 좋다고 판단했다.

>[!tip] 정리
>마킹이 들어올 때마다 DB에 바로 쓰면 너무 많은 쿼리가 나가고 레이턴시가 커져서, 현재 마킹 상태는 Redis Hash에 올려두고 WebSocket으로만 실시간 반영해준다. 이후에 워크스페이스 저장/종료/주기 플러시 시점에 Redis에 모여 있는 마킹 상태를 한 번에 DB로 flush해서, Redis는 실시간 상태 캐시, DB는 스냅샷/영속 저장소 역할을 하도록 분리했다.



## 2.  Redis 원자성 적용하기 

### 2.1.  배경 
여러 이벤트가 동시에 발생할 수 있기 때문에, 마킹 저장 로직은 **레이스 컨디션에 매우 취약**하다.

```js
// ❌문제의 코드 
await client.del(key);      // 1️⃣ 기존 데이터 삭제
await client.hSet(key, ...);// 2️⃣ 새 데이터 삽입
```
- 이 두 명령은 따로 실행되어 중간에 경쟁 조건이 발생할 수 있다.


### 2.2.  원자성(Atomic)이란?

> 어떤 작업이 더 이상 쪼개질 수 없는 한다의 단위로 간주되는 성질 

이는 멀티스레드 환경에서 매우 중요한 성질이다.
- 여러 스레드가 동시에 공유 자원 접근 시 문제가 발생할 수 있는데,
- 원자성을 보장한다면 이러한 문제를 해결할 수 있다.
- *어떻게❓*
	- 여러 작업을 하나의 분할 불가능한 연산으로 묶어 연산 완료할 때까지 다른 스레드의 접근을 막는다 by 세마포어/뮤텍스 

### 2.3.  Redis 트랜잭션(MULTI/EXEC)으로 원자성 해결
> 여러 명령을 하나의 원자적 연산으로 묶기 위해 **MULTI / EXEC** 트랜잭션을 도입

Redis는 단일 스레드 구조이지만, **명령어 단위의 원자성만 보장**한다.

```js
const pipeline = client.multi(); // 트랜잭션 시작

pipeline.del(key);
pipeline.hSet(key, hashObject);
pipeline.expire(key, ttl);

await pipeline.exec(); // 한꺼번에 실행
```
이 방식을 통해 
- 삭제 ➡ 삽입 ➡ TTL 설정까지 하나의 원자적 처리
- 마킹 데이터의 **일관성(Consistency)** 보장


## 3.  Redis Hash 구조 설계해보기
참고 : [인파 Redis 구조](https://inpa.tistory.com/entry/REDIS-%F0%9F%93%9A-%EB%8D%B0%EC%9D%B4%ED%84%B0-%ED%83%80%EC%9E%85Collection-%EC%A2%85%EB%A5%98-%EC%A0%95%EB%A6%AC#redis_-_bitmaps)

> [!TIP] 하나의 KEY 아래에 작은 JSON 객체(hash)를 넣는 구조 

### 3.1.  Hash구조로 저장하려는 이유 
>[!info] 참고 
>이번 프로젝트에서는 Hash구조만 사용하지않고 일부 데이터는 List형식으로도 일부 저장하기는 하지만 **주로 Hash 구조로 저장**한다 
- 하나의 Redis Key 아래에 **여러 개의 POI/PlanDay 상태를 필드(field) 단위로 저장**
- 워크스페이스 단위로 상태 조회가 빠르고 직관적
- 부분 업데이트가 가능하여 WebSocket 기반 실시간 협업에서 효율적


### 3.2.  Redis의 Hash 구조 
```js
// 구조 
hset(key, field, value)
// 예시 
await client.hSet(key, poiConnection.nextPoiId, JSON.stringify(poiConnection));
```
![Pasted image 20251105134009.png](/img/user/supporter/image/Pasted%20image%2020251105134009.png)
- hash-key 안에 내부 subkey를 이용하는 자료구조 
- key : Redis 기준 최상위 키 
- filed : Hash 안의 서브 키 
- value : 문자열 값 

| 인자         | 의미         | 예시 값                                           | Redis 내부 표현    |
| ---------- | ---------- | ---------------------------------------------- | -------------- |
| 1. `key`   | 해시 이름      | `"poi_connection:plan_day_123"`                | 해시 테이블 이름      |
| 2. `field` | 해시 내부의 식별자 | `"nextPoiId-abc123"`                           | Hash의 key 부분   |
| 3. `value` | 실제 데이터     | `{"distance":100,"duration":60}` (stringified) | Hash의 value 부분 |

```json
Key: poi-connection:{workSpaceId}
Field: {planDayId}
Value: JSON.stringify(poiConnections)[]
```

```json
Key: poi-connection:{planDayId}
Field: {poiConnectionId}
Value: JSON.stringify(poiConnection)
```


### 3.3.  명령어 

1. `hget(key, field)` - 특정 필드 하나만 가져오기 
2. `hgetAll(key)` - Hash 전체 다 가져오기
3. `hset(key, field)` - 특정 필드 값 추가/수정
4. `hdel(key, field)` - 특정 필드 삭제 

