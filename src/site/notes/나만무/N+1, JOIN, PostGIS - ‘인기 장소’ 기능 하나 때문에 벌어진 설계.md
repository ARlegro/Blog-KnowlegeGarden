---
{"dg-publish":true,"permalink":"/나만무/N+1, JOIN, PostGIS - ‘인기 장소’ 기능 하나 때문에 벌어진 설계/","noteIcon":"","created":"2025-12-03T14:53:06.863+09:00","updated":"2025-12-09T17:50:25.213+09:00"}
---



## 1.  요구사항

지도 화면에서 특정 뷰포트(지도 경계, Bounds) 안에 있는 장소를 가져오는 API `getPlacesInBounds`가 이미 있었다. 여기에 추가 요구사항이 붙었다.

> 범위 내 사용자가 장소들을 보고 있을 때, 서비스 내 사용자들에게 인기 있는 장소를 보여줄 수 있어야 한다.”


내 서비스에서는 이미 **RabbitMQ 기반 비동기 파이프라인**으로 사용자 행동 로그를 모으고 있다.  
이 중에서 아래 두 이벤트를 “인기”의 기준으로 삼기로 했다.
- `POI_MARK` : 사용자가 지도에서 장소를 직접 찍어 마킹한 이벤트
- `POI_SCHEDULE` : 사용자가 여행 일정에 장소를 실제로 추가한 이벤트
    
즉, **특정 장소에 대해 위 두 이벤트가 많이 발생할수록 “인기 있는 장소”로 판단**하고 싶었다.


문제는,
- `places` 테이블에는 인기 여부에 대한 컬럼이 없고
- 행동 로그는 별도의 `user_behavior_events` 테이블에 쌓이고 있다는 점이었다.<br>![Pasted image 20251121151936.png](/img/user/supporter/image/Pasted%20image%2020251121151936.png)

그래서 **기존 `getPlacesInBounds` 조회 로직 안에서, 각 장소의 인기도 점수를 어떻게 붙일지**가 핵심 고민 포인트였다.
  
```ts

  async getPlacesInBounds(dto: GetPlacesReqDto): Promise<GetPlacesResDto[]> {
    const {
      southWestLatitude,
      southWestLongitude,
      northEastLatitude,
      northEastLongitude,
    } = dto;

    const t0 = performance.now();
    const places: Place[] = await this.placeRepo
      .createQueryBuilder("p")
      .where(
        `ST_Intersects(
        p.location, ST_MakeEnvelope(:southWestLongitude, :southWestLatitude, :northEastLongitude, :northEastLatitude))`,
        {
          southWestLongitude,
          southWestLatitude,
          northEastLongitude,
          northEastLatitude,
        }
```

## 2.  어떻게 해결해야하는가?
![Pasted image 20251121152302.png](/img/user/supporter/image/Pasted%20image%2020251121152302.png)
RabbitMQ로 수집된 행동 이벤트는 `user_behavior_events` 테이블에 저장된다.  
여기서 **특정 Bounds 안에 있는 장소들에 대한 “인기도 점수”를 함께 가져오는** 여러 가지 방법을 고민했다. 어떻게 효율적으로 할 수 있을까?

### 2.1.  고려 1 - Place하나씩 Event Count (전형적인 N+1패턴)
가장 먼저 떠올릴 수 있는 방식은 단순하다.
1. 기존 `getPlacesInBounds`로 Bounds 안의 `places` 목록을 먼저 조회
2. 조회된 각 `place.id`에 대해, `user_behavior_events`에서 `COUNT(*)`를 한 번씩 수행
3. 그 결과를 조합해서 응답

💢문제 - N+1
- 장소가 100개 나오면 → 쿼리는 1 + 100번 = 101번 호출
- 데이터가 많아질수록 **대표적인 N+1 문제**로 이어짐
- 특히 행동 로그 테이블은 시간이 지날수록 계속 커지므로 이런 패턴은 장기적으로 유지가 어렵다고 판단했다.


### 2.2.  고려 2. 두 번의 독립 쿼리로 해결 (JOIN 회피)
두 번째로 생각한 방식은 **JOIN을 완전히 피하는 방식**이다.
1. 먼저 Bounds 안에 있는 `place` 목록을 가져온다.
2. 그 `place.id` 리스트를 IN 조건으로 묶어, 한 번에 인기 점수를 가져온다.

```sql
-- 1) Bounds 안의 장소 조회
-- ...
-- 2) 그 id들에 대한 인기도 집계
SELECT place_id, COUNT(*) AS popularity_score
FROM user_behavior_events
WHERE place_id IN (...)
  AND event_type IN ('POI_MARK', 'POI_SCHEDULE')
GROUP BY place_id;
```
*✅장점*
- 최소 쿼리 수를 2번으로 고정
- N + 1 문제 사라짐 

*💢단점 - 유지보수 귀찮*
- 카테고리/필터 조합을 바꾸기 어렵다
	- 만약 필터가 더 늘어난다면 두 개의 쿼리에 동일한 필터 로직을 유지해야 함 
	- 이는 로직 중복과 유지보수 비용증가

실시간으로 요구사항이 자주 바뀌는 프로젝트 특성상 유지 보수 및 확장성을 위해 이 방식 말고 다음 방식을 택했다.

### 2.3.  고려 3. JOIN + GROUP BY로 한 번에 해결
> 위의 2가지 방법의 한계로 인해 **JOIN을 사용해서 한 번에 집계하는 구조**로 방향을 잡았다.

```SQL
SELECT p.*, COUNT(ube.id) as popularity_score
  FROM places p
  LEFT JOIN user_behavior_events ube
    ON ube.place_id = p.id
    AND ube.event_type IN ('POI_MARK', 'POI_SCHEDULE')
  WHERE ST_DWithin(p.location, ST_MakePoint($lng, $lat)::geography, $radius)
    -- 또는 bounds 조건
  GROUP BY p.id
  ORDER BY popularity_score DESC
  LIMIT $n;
```

*선택 이유* 
1. **한 번의 쿼리로 해결** 
2. **카테고리/필터 조합에 유리**
3. **PostGIS + 인덱스 덕분에 JOIN이 생각보다 비싸지 않을 것 같다**
	- JOIN에 거부감이 있었던 이유는 과거에 대용량 테이블끼리 JOIN하다가 쿼리가 터지는 경험을 했기 때문인데, 이번 케이스는 구조가 좀 달랐다.
	- `places.location` 에 **GiST 공간 인덱스**가 걸려 있어 후보군을 강하게 줄일 수 있고
	- `user_behavior_events.place_id` 에도 인덱스가 있어서 JOIN 시 랜덤 스캔이 아니라 인덱스 기반 탐색으로 처리된다.
	- 즉, 무작정 JOIN이 아니라 필터링이 많이 된 결과에 대해 인덱스 기반 JOIN이라 실제 성능 부담이 크지 않을 것으로 판단

```ts
    const rawPlaces = await this.placeRepo
      .createQueryBuilder('p')
      .leftJoin(
        'user_behavior_events',
        'ube',
        'ube.place_id = p.id AND ube.event_type IN (:...eventTypes)',
        { eventTypes: ['POI_MARK', 'POI_SCHEDULE'] },
      )
      .where(
        `ST_Intersects(
          p.location,
          ST_MakeEnvelope(:swLng, :swLat, :neLng, :neLat, 4326)::geography
        )`,
        {
          swLng: southWestLongitude,
          swLat: southWestLatitude,
          neLng: northEastLongitude,
          neLat: northEastLatitude,
        },
      )
      .select('p.id', 'id')
      .addSelect('p.title', 'title')
      .addSelect('p.address', 'address')
      .addSelect('p.category', 'category')
      .addSelect('p.summary', 'summary')
      .addSelect('p.image_url', 'image_url')
      .addSelect('p.longitude', 'longitude')
      .addSelect('p.latitude', 'latitude')
      .addSelect('COUNT(ube.id)', 'popularity_score')
      .groupBy('p.id')
      .limit(20)
      .getRawMany<{
        id: string;
        title: string;
        address: string;
        category: string;
        summary: string | null;
        image_url: string | null;
        longitude: number;
        latitude: number;
        popularity_score: string;
      }>();
```
