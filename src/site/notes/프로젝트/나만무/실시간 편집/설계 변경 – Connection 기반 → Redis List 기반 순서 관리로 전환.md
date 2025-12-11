---
{"dg-publish":true,"permalink":"/프로젝트/나만무/실시간 편집/설계 변경 – Connection 기반 → Redis List 기반 순서 관리로 전환/","noteIcon":"","created":"2025-12-03T14:53:06.783+09:00","updated":"2025-12-11T14:35:04.046+09:00"}
---


> POI 간 연결(Connection) 테이블 기반 순서 관리의 복잡도와 성능 문제를 해결하기 위해, Redis List + Hash 조합으로 일정 순서를 관리하도록 아키텍처를 전면 교체했다.



## 1.  기존 설계 : Connection 테이블 기반 순서 관리


### 1.1.  기존 설계 구조
기존 구조는 POI 간의 **연속된 관계(prev, next)** 를 모두 Connection 테이블에 저장하고 있었다.

A → B → C → D → E
(5개 POI → 4개의 Connection)


![Pasted image 20251109131331.png](/img/user/supporter/image/Pasted%20image%2020251109131331.png)

### 1.2.  문제 
![Pasted image 20251109132622.png](/img/user/supporter/image/Pasted%20image%2020251109132622.png)

사용자가 순서를 조금만 바꿔도, 여러 Connection을 수정해야 했다.
- 두 개의 장소를 변경했을 때 커넥션 정보를 최대 4개 변경이 必
- 일정에 포함된 커넥션들을 다 지우고 다시 만드는 것은 싫다.
- Drag & Drop 변경 시 Connection diff 계산 비용 증가

>[!QUESTION]  커넥션 테이블을 다 지우고 확정된 poi에 대한 테이블을 만들어 순서를 부여하는 것은❓
>No, Poi는 소요시간 및 거리에 대한 정보를 저장할 수 없기 때문에 커넥션 정보 테이블이 필요하다 

즉, 핵심 문제 = 순서는 매우 자주 변하지만, Connection 구조는 조작 비용이 너무 크다


## 2.  초기 개선 아이디어 - diff 기반 connection 재계산

**초기에는 프론트에서 전체 순서를 받아 diff를 계산**해, 바뀐 Connection만 업데이트하는 전략을 고민했다.

```js
// 변경 전
oldOrder = ["A", "B", "C", "D", "E"]

// 사용자가 드래그 후
newOrder = ["A", "D", "C", "B", "E"]
```
1. 연속된 쌍을 만들고 
2. 새로운 연속 쌍을 만들고
3. 삭제할 커넥션 : `oldPairs - newPairs`
4. **추가할 커넥션** = `newPairs - oldPairs`

```SQL
-- IN 절로 일괄 삭제하고....변경분 추가하고....근데 너무 복잡하다...
DELETE FROM poi_connection
WHERE plan_day_id = $1
AND (prev_poi_id, next_poi_id) IN ( VALUES
	(
		($2, $3), 
		($4, $5),
		($6, $7)
	)
)
```


이 방식은 DB 최소 변경량이라는 장점이 있지만, **Connection 모델 자체의 복잡성**과 여전히 JOIN성능 비용이 걱정됐다. 


## 3.  최종 해결 아이디어 - Redis List 기반 순서 관리 아키텍처
> 결국 Connection 테이블을 제거하고, Poi테이블의 sequence필드를 두고**순서 자체를 Redis List로 관리하는 구조**로 전환


테이블 1개(Poi)두고 상태(Marked, Scheduled)필드와 순서 필드 추가 
- 순서 필드 : 특정 plan_day_id의 Scheduled 된 poi들의 순서 
- 정규화가 덜 됐다고 볼 수 있지만 유지보수 및 성능 측면(join 줄이는)에서 유리 
- 또한 드래그앤드랍 처리시 간단 

### 3.1.  Redis 관리 구조 (List + Hash)
Redis List - POI 순서 고나리 
- 각 plan_day의 POI 순서를 배열 형태로 유지
- Drag & Drop → 프론트가 보내는 poi_ids 배열을 그대로 Redis List에 재구성 
- 순서 = List 인덱스
    
Redis Hash - POI 상세 정보 관리 
- Workspace별 POI 상세 정보 캐싱
- sequence 필드는 List 인덱스와 동기화 ➡DB I/O 감소소

### 3.2.  실제 코드 동작(REORDER 이벤트)
> 실제 사용자가 일정을 바꿀 때 `REORDER`이벤트를 발생시키는데 이 때 어떻게 처리하는지 알아볼 것

*프론트가 보내는 내용*
```JSON
{
  "workspaceId": "...",
  "planDayId": "...",
  "poiIds": ["poi-A", "poi-D", "poi-C", "poi-B", "poi-E"]
}
```

*서버의 처리 흐름*
1. *Redis List 전체 재구성*
	```TS
	await this.poiCacheService.reorderScheduledPois(workspaceId, planDayId, poiIds);
	```
2. *Hash에 필드 sequence 필드 업데이트*
	- List 인덱스 = POI 새로운 순서 값
	- Note : 일정에서 제거되는거는 `REORDER`이벤트와 다른 것 
3. *모든 클라이언트에 브로드캐스트*

### 3.3.  Redis 기반 구조 리팩토링 장점

1. *성능 개선 -JOIN 불필요 + O(1)*
	- 매번 일정을 드래그앤드랍으로 업데이트할 때마다 JOIN을 활용할 필요가 없어진다.
	- 순서 변경 시 O(1) 수준으로 처리 Cuz 리스트 재구성 
	  
2. *단순성 및 유지보수성 향상*
	- POI 순서 관리 로직이 극적으로 단순해졌다.
	- 이전에 고려했단 diff 계산 / 여러 connection 업데이트 같은 고비용 작업이 제거됐기 때문이다.

> DB설계 입장에서 필드가 책임분리가 안되서 정규화가 부족해보이지만 **성능 + 단순성이 장점**인 것 같다.
