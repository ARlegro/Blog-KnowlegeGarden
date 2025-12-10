---
{"dg-publish":true,"permalink":"/프로젝트/나만무/AI-Agent/AI Agent 타임아웃 - 공간 인덱싱 적용/","noteIcon":"","created":"2025-12-06T12:52:49.596+09:00","updated":"2025-12-10T13:29:12.554+09:00"}
---


### 0.1.  목차

- [1.  타임아웃 발생](#1.%20%20%ED%83%80%EC%9E%84%EC%95%84%EC%9B%83%20%EB%B0%9C%EC%83%9D)
- [2.  PostGIS + GiST 공간 인덱스 도입](#2.%20%20PostGIS%20+%20GiST%20%EA%B3%B5%EA%B0%84%20%EC%9D%B8%EB%8D%B1%EC%8A%A4%20%EB%8F%84%EC%9E%85)
	- [2.1.  PostGIS 설치](#2.1.%20%20PostGIS%20%EC%84%A4%EC%B9%98)
	- [2.2.  공간 쿼리 성능 최적화](#2.2.%20%20%EA%B3%B5%EA%B0%84%20%EC%BF%BC%EB%A6%AC%20%EC%84%B1%EB%8A%A5%20%EC%B5%9C%EC%A0%81%ED%99%94)
- [3.  자세한 개념 공부](#3.%20%20%EC%9E%90%EC%84%B8%ED%95%9C%20%EA%B0%9C%EB%85%90%20%EA%B3%B5%EB%B6%80)
- [4.  공간 인덱싱을 통한 성능 개선](#4.%20%20%EA%B3%B5%EA%B0%84%20%EC%9D%B8%EB%8D%B1%EC%8B%B1%EC%9D%84%20%ED%86%B5%ED%95%9C%20%EC%84%B1%EB%8A%A5%20%EA%B0%9C%EC%84%A0)
	- [4.1.  Haversine 공식 사용](#4.1.%20%20Haversine%20%EA%B3%B5%EC%8B%9D%20%EC%82%AC%EC%9A%A9)
	- [4.2.  PostGIS 공간 인덱스 사용](#4.2.%20%20PostGIS%20%EA%B3%B5%EA%B0%84%20%EC%9D%B8%EB%8D%B1%EC%8A%A4%20%EC%82%AC%EC%9A%A9)
- [5.  지도 뷰포트(Zoom/Drag) 조회 성능 안정화](#5.%20%20%EC%A7%80%EB%8F%84%20%EB%B7%B0%ED%8F%AC%ED%8A%B8(Zoom/Drag)%20%EC%A1%B0%ED%9A%8C%20%EC%84%B1%EB%8A%A5%20%EC%95%88%EC%A0%95%ED%99%94)
	- [5.1.  Previw - 결론](#5.1.%20%20Previw%20-%20%EA%B2%B0%EB%A1%A0)
	- [5.2.  공통 조건](#5.2.%20%20%EA%B3%B5%ED%86%B5%20%EC%A1%B0%EA%B1%B4)
	- [5.3.  💢 기존 방식 – `Between` 연산으로 위경도 필터링](#5.3.%20%20%F0%9F%92%A2%20%EA%B8%B0%EC%A1%B4%20%EB%B0%A9%EC%8B%9D%20%E2%80%93%20%60Between%60%20%EC%97%B0%EC%82%B0%EC%9C%BC%EB%A1%9C%20%EC%9C%84%EA%B2%BD%EB%8F%84%20%ED%95%84%ED%84%B0%EB%A7%81)
	- [5.4.  ✅ 공간인덱싱 타게 한 연산](#5.4.%20%20%E2%9C%85%20%EA%B3%B5%EA%B0%84%EC%9D%B8%EB%8D%B1%EC%8B%B1%20%ED%83%80%EA%B2%8C%20%ED%95%9C%20%EC%97%B0%EC%82%B0)
	- [5.5.  관찰 결과](#5.5.%20%20%EA%B4%80%EC%B0%B0%20%EA%B2%B0%EA%B3%BC)
	- [5.6.  정리](#5.6.%20%20%EC%A0%95%EB%A6%AC)
	- [5.7.  번외 - 성능 분산 원리](#5.7.%20%20%EB%B2%88%EC%99%B8%20-%20%EC%84%B1%EB%8A%A5%20%EB%B6%84%EC%82%B0%20%EC%9B%90%EB%A6%AC)


> 공간 인덱싱(PostGIS)을 도입하게 되기까지의 기록이다.

---
## 1.  타임아웃 발생 
AI 기반 여행 추천 기능을 위해 위도/경도 좌표 기반으로 근처 맛집, 숙소, 관광지 등을 추천하는 툴들을 만들기 시작했다.
근처를 검색하면 되니까 대충 거리 계산 공식 검색 후 Haversine공식을 사용했다.
그런데 AI Agent가 실행되면서 TimeOut 경고를 내기 시작했다.💢
```bash 
> > Entering new AgentExecutor chain...

Invoking: `recommend_nearby_places` with `{'location_name': '해운대', 'category': '음식', 'radius_km': 5, 'limit': 10}`
responded: [{'type': 'tool_use', 'name': 'recommend_nearby_places', 'input': {'location_name': '해운대', 'category': '음식', 'radius_km': 5, 'limit': 10}, 'id': 
'tooluse_Aks9VXblStilnakW5oKtcw'}]

장소 추천 중 에러 발생: timed out
Invoking: `recommend_nearby_places` with `{'location_name': '해운대', 'category': '음식'}`
responded: [{'type': 'text', 'text': '죄송합니다. 해운대 주변 맛집 정보를 조회하는 데 문제가 있어 결과를 제공하지 못했습니다. 다시 시도해 보겠습니다.'}, {'type': 'tool_use', 
'name': 'recommend_nearby_places', 'input': {'location_name': '해운대', 'category': '음식'}, 'id': 'tooluse_ZYKlVDiAR66UwZgZMJiDhA'}] 
```
```bash
WARNING:  WatchFiles detected changes in 'app/tools/place_tool.py'. Reloading...
장소 추천 중 에러 발생: timed out다시 시도했지만 여전히 문제가 있네요. 해운대 주변 맛집 정보를 직접 찾아보겠습니다.

해운대는 부산의 대표적인 관광지로, 다양한 맛집들이 있습니다. 해운대 해수욕장 근처에는 신선한 해산물과 부산 토속 음식을 맛볼 수 있는 식당들이 많습니다. 특히 해운대 해변가에 위치한 '해운대 해변 맛집'과 '해운대 해수욕장 맛집'은 유명합니다. 이곳에서는 회, 돌솥비빔밥, 해물탕 등 부산 특산물을 맛볼 수 있습니다. 또한 해운대 마린시티 인근에도 다양한 레스토랑과 카페가 있어 분위기 있게 식사를 즐길 수 있습니다. 이 외에도 해운대 구남로와 중동 일대에도 맛집들이 많이 있으니 둘러보시면 좋을 것 같습니다.

> Finished chain.
{'input': '부산 해운대 주변 맛집 추천해줘', 'chat_history': [], 'session_id': 'test-session_id', 'output': "다시 시도했지만 여전히 문제가 있네요. 해운대 주변 맛집 정보를 직접 찾아보겠습니다.\n\n해운대는 부산의 대표적인 관광지로, 다양한 맛집들이 있습니다. 해운대 해수욕장 근처에는 신선한 해산물과 부산 토속 음식을 맛볼 수 있는 식당들이 많습니다. 특히 해운대 해변가에 위치한 '해운대 해변 맛집'과 '해운대 해수욕장 맛집'은 유명합니다. 이곳에서는 회, 돌솥비빔밥, 해물탕 등 부산 특산물을 맛볼 수 있습니다. 또한 해운대 마린시티 인근에도 다양한 레스토랑과 카페가 있어 분위기 있게 식사를 즐길 수 있습니다. 이 외에도 해운대 구남로와 중동 일대에도 맛집들이 많이 있으니 둘러보시면 좋을 것 같습니다.", 'intermediate_steps': [(ToolAgentAction(tool='recommend_nearby_places', tool_input={'location_name': '해운대', 'category': '음식', 'radius_km': 5, 'limit': 10}, log="\nInvoking: `recommend_nearby_places` with `{'location_name': '해운대', 'category': '음식', 'radius_km': 5, 'limit': 10}`\nresponded: [{'type': 'tool_use' 
....
```

AI Agent가 실행되면서 타임아웃 경고를 내기 시작했다.💢  
해운대·강남역 주변처럼 데이터가 많은 지역을 반경 검색할 때, 단일 요청이 **10초 이상 걸리는 구간**이 생겼고, 에이전트가 같은 툴을 여러 번 재시도하면서 **전체 응답 시간이 수십 초까지 늘어난 뒤 타임아웃**에 걸리는 패턴이 반복됐다.

---
## 2.  PostGIS + GiST 공간 인덱스 도입 

공간 데이터 계산을 DB에 맡기기로 했다.

### 2.1.  PostGIS 설치 
- DB에 설치
- 패키지에 설치
	```BASH
	uv pip install geoalchemy2
	```


장소 데이터에 geography 정보가 추가

---
### 2.2.  공간 쿼리 성능 최적화

```sql
-- GiST 인덱스 생성 (공간 쿼리 성능 최적화)
 CREATE INDEX IF NOT EXISTS idx_places_location ON places USING GIST(location);
```
  성능 개선 효과
  - 기존: Haversine 공식으로 Full Table Scan (느림, 30초+ 타임아웃)
  - 개선 후: PostGIS GiST 인덱스로 공간 검색 (예상: 0.1~0.5초)


---
## 3.  자세한 개념 공부 
[[프로젝트/나만무/DB관련/PostGIS와 GiST 인덱싱\|PostGIS와 GiST 인덱싱]]


---
## 4.  공간 인덱싱을 통한 성능 개선 

> 요구 사항 : “현재 위치 기준 반경 N km 안의 장소를 찾는다”
> 개선 결과 : 기존 방식 대비 **최대 181배 성능 향상** 수치

*분석*
- PostgreSQL에 약 **21,929개(전국 장소 데이터)** 가 저장된 상태
- `강남역 / 해운대` 주변 장소를 반경 기반으로 조회하는 성능을 측정(애플리케이션부터)

| 케이스            | PostGIS (공간 인덱스) | Haversine (인덱스 미사용) | 성능 차이   |     |
| -------------- | ---------------- | ------------------- | ------- | --- |
| 강남역 5km (10개)  | 0.37초            | 16.47초              | 44배 빠름  |     |
| 강남역 10km (20개) | 0.94초            | 13.11초              | 14배 빠름  |     |
| 해운대 5km (10개)  | 0.07초            | 12.44초              | 181배 빠름 |     |
|                |                  |                     |         |     |
- DB에 저장된 Place 개수 : 21,929개 

---
### 4.1.  Haversine 공식 사용 
> 애플리케이션 레벨의 Haversine 거리 계산으로 구현한 첫 번째 버전

**Haversine 공식❓** 
- 지구상의 두 지점 사이의 최단거리를 계산하는 데 사용하는 공식 
- **직선거리가 아닌 실제 곡면상의 최단거리**를 구함 (지구가 둥근 성질을 반영)
- 주변 POI 검색 

```PYTHON
        R = 6371000  # 지구 반경 (미터)
        
        # 라디안으로 변환
        lat1_rad = math.radians(lat1)
        lat2_rad = math.radians(lat2)
        delta_lat = math.radians(lat2 - lat1)
        delta_lon = math.radians(lon2 - lon1)
  
        # Haversine 공식
        a = math.sin(delta_lat / 2) ** 2 + \
            math.cos(lat1_rad) * math.cos(lat2_rad) * \
            math.sin(delta_lon / 2) ** 2
        c = 2 * math.asin(math.sqrt(a))
```

*💢문제점(기술적 한계)* - 계산은 애플리케이션 
1. *Full Table Scan + 애플리케이션 필터링*
	- 지구의 곡률을 반영한 거리 검색은 DB에서 할 수 없으므로 DB에서 모든 장소를 가져온 뒤 Haversine 공식을 사용해야 한다. 
	- 이로 인해 Full Table Scan이 발생하고 메모리/CPU를 애플리케이션이 부담지게 된다.
	  
2. *O(N) 거리 계산에 따른 CPU 병목*
	- 데이터가 늘어날수록 성능이 선형으로 나빠지는 구조 
	  
3. *DB 최적화 기능 사용 불가*
	- 곡률 계산으로 인해 거리 계산을 애플리케이션 레벨에서 담당하기 때문의 DB의 인덱스, 쿼리 플래너 등의 DB Optimize 기능을 사용하지 못한다.

---
### 4.2.  PostGIS 공간 인덱스 사용 
![Pasted image 20251118224442.png](/img/user/supporter/image/Pasted%20image%2020251118224442.png)
Haversine 방식의 한계를 극복하기 위해 좌표를 단순 `FLOAT` 컬럼이 아닌 PostGIS의 **`geography` 타입 + 공간 인덱스(GiST)** 를 도입해 DB 레벨에서 거리 계산 및 필터링을 수행하도록 변경
![Pasted image 20251118200420.png](/img/user/supporter/image/Pasted%20image%2020251118200420.png)
```sql 
-- geography 컬럼 추가 
ALTER TABLE places
	ADD COLUMN location geography(Point, 4326);

-- 기존 latitude/longitude에서 coord로 변환
UPDATE places
SET location = ST_SetSRID(ST_MakePoint(longitude, latitude), 4326)::geography;

-- 공간 인덱스 생성 (GiST)
CREATE INDEX idx_places_coord_gist
ON places USING GIST (location);
```

1. *빠른 후보 결정* 
	- `ST_DWithin` + GiST 인덱스를 사용하면, **인덱스를 통해 반경 안에 있을 가능성이 있는 후보만 빠르게 추려낸 후 실제 거리를 계산하고 애플리케이션에 반환** 
	  
2. *네트워크 트래픽 부담 감소*
	- DB 내부에서 거리 계산 + 정렬까지 처리하기 때문에 애플리케이션과 DB사이의 네트워크 트래픽은 필요한 결과 N개만 전송된다.
	- 거리 계산과 정렬이 모두 DB 내부에서 수행되므로



> *생각*
> - *처음 생각*  "위도/경도만 있으면 애플리케이션에서 거리 공식으로 처리하면 되겠지”
> - *하면서 든 생각* : 장소 데이터가 2만개 밖에 없음에도 너무 오랜 시간이 걸린다. 여기서 데이터를 더 넣는 순간 엄청난 병목 발생 



---
## 5.  지도 뷰포트(Zoom/Drag) 조회 성능 안정화
> 소켓으로 줌 인/아웃 및 지도 이동(focus) 이벤트가 발생할 때마다,  **현재 뷰포트(bounding box) 안에 있는 장소를 조회**하는 로직의 성능을 비교했다.

### 5.1.  Previw - 결론

**`ST_Intersects`** 및 **`ST_MakeEnvelope`** 함수를 GiST 인덱스와 함께 사용하도록 최적화

| **구분**     | **기존 방식 (Latitude/Longitude BETWEEN 사용)** | **개선 방식 (ST_Intersects + GiST 인덱스)**    |
| ---------- | ----------------------------------------- | --------------------------------------- |
| **성능 편차**  | 뷰포트가 클 때 **900ms까지 튀는 스파이크** 발생           | **대부분 20~80ms** 구간에 안정화                 |
| **사용자 체감** | 성능 편차가 커서 예측 불가능하고 불안정                    | **평균 응답 및 분산이 크게 줄어** 일관되고 예측 가능한 성능 확보 |
```ts
  async getPlacesInBounds(dto: GetPlacesReqDto): Promise<GetPlacesResDto[]> {
    const {
      southWestLatitude,
      southWestLongitude,
      northEastLatitude,
      northEastLongitude,
    } = dto;
  
    const places: Place[] = await this.placeRepo
      .createQueryBuilder('p')
      .where(
	      // 여기 
        `ST_Intersects(
      p.location, ST_MakeEnvelope(:southWestLongitude, :southWestLatitude, :northEastLongitude, :northEastLatitude))`,
        {
          southWestLongitude,
          southWestLatitude,
          northEastLongitude,
          northEastLatitude,
        },
      )
      .limit(20)
      .getMany();
    return places.map((place) => GetPlacesResDto.from(place));
  }
```

*뷰포트(Focus) 기반 조회 성능 - Between vs ST_Intersects*
- 지도 Focus 이벤트마다 
- **기존 Between 방식**
    - 작은 범위 / 기존 조회 근처: 20~30ms 수준으로 그럭저럭 양호
    - 뷰포트를 크게 움직이거나, 완전히 다른 지역으로 점프, 아주 넓은 영역 조회 시 **수백 ms ~ 900ms까지 튀는 스파이크** 관찰
    - 성능 편차가 크고 예측 가능성이 낮아지는 문제가 존재.
        
- **PostGIS ST_Intersects + GiST 인덱스**
    - 대부분 케이스: **20~80ms 사이**에 응답
    - 좁은 뷰포트에서는 **Between과 거의 동일한 속도(20~40ms)**
    - 근처로 살짝 이동하는 일반적인 줌/드래그 상황에서는 **거의 항상 20~40ms 선에 수렴**
    - 아주 넓은 범위를 한 번에 조회하는 극단 케이스에서만 수백 ms 구간이 일부 존재하나, Between처럼 900ms까지 폭발하는 케이스가 크게 줄음
    - **평균 응답 및 분산이 크게 줄어 체감 성능이 안정화**



---

### 5.2.  공통 조건
- 트리거: 클라이언트에서 `PLACE_FOCUS` 이벤트 발생 시마다 서버에서 DB 조회
- 입력: `southWestLatitude`, `northEastLatitude`, `southWestLongitude`, `northEastLongitude`
- 출력: 현재 bounds 안의 `Place` 최대 20개
- 측정 방식: NestJS 서비스 레벨에서 `performance.now()`로  
    **DB 조회 전후 시간을 측정**해 millisecond 단위로 로깅

```ts
const t0 = performance.now();
// DB 조회
const t1 = performance.now();

console.log(
  `[getPlacesInBounds] ... 걸린 시간: ${(t1 - t0).toFixed(4)} ms`,
);
```

---
### 5.3.  💢 기존 방식 – `Between` 연산으로 위경도 필터링

*기존 - Between 연산 사용*
```js
    const t0 = performance.now();
    const places: Place[] = await this.placeRepo.find({
      where: {
        latitude: Between(southWestLatitude, northEastLatitude),
        longitude: Between(southWestLongitude, northEastLongitude),
      },
      take: 20,
    });
    const t1 = performance.now();
    console.log(
      `[getPlacesInBounds] Between 연산으로 걸린 시간: ${(t1 - t0).toFixed(4)} ms`,
    );
    console.log('가져온 개수 = ', places.length);
```

TypeORM이 만든 실제 쿼리 
```sql
SELECT ...
FROM "places" "Place"
WHERE
  ("Place"."latitude" BETWEEN $1 AND $2)
  AND ("Place"."longitude" BETWEEN $3 AND $4)
LIMIT 20;
```


✔결과 분석 
- **근처에서 살짝만 이동/줌** → 대체로 **20~30ms 수준**으로 꽤 빠르게 응답
- **지도를 크게 이동하거나, 완전히 다른 지역으로 점프**하면
    - **수백 ms 단위까지 튀는 구간 존재 (e.g. 395ms, 631ms, 947ms 등)**
    - 특히 **처음 해당 영역을 조회할 때** 지연이 크게 발생
        
요약
- **작은 범위 / 기존 조회 근처**: 괜찮음 (20ms 안팎)
- **범위가 넓거나, 처음 가는 영역일 때**: **최대 900ms 근처까지 지연 발생**

이는 latitude/longitude 각각에 대한 단순 범위 검색이라,
- 인덱스가 있어도 **선택도가 떨어지는 큰 범위**에서는
- 내부적으로 **많은 row를 스캔**하게 되고,
- 그 결과, 뷰포트가 크게 움직이는 경우 성능이 급격히 나빠지는 모습이다.

---
### 5.4.  ✅ 공간인덱싱 타게 한 연산
```JS
    const t0 = performance.now();
    const places: Place[] = await this.placeRepo
      .createQueryBuilder('p')
      .where(
        `ST_Intersects(
      p.location, ST_MakeEnvelope(:southWestLongitude, :southWestLatitude, :northEastLongitude, :northEastLatitude))`,
        {
          southWestLongitude,
          southWestLatitude,
          northEastLongitude,
          northEastLatitude,
        },
      )
      .limit(20)
      .getMany();
```

```SQL
SELECT ...
FROM "places" "p"
WHERE ST_Intersects(
  p.location,
  ST_MakeEnvelope($1, $2, $3, $4)
)
LIMIT 20;
```

---
### 5.5.  관찰 결과

- **대부분의 케이스**:
    - **20~80ms 사이**에서 안정적으로 응답
    - 근처로 살짝 이동하면서 포커스를 옮기는 경우,  
        거의 항상 **20~40ms 안쪽**에 수렴
        
- **아주 넓은 범위를 한 번에 보는 경우 / 데이터가 거의 없는 지역**:
    - **수백 ms까지 튀는 구간도 여전히 존재 (400ms대 스파이크 관찰)**
    - 특히 **데이터가 희박한 영역을 넓게 잡고 조회**할 때, 1~몇 개만 나와도 시간은 길어질 수 있음

즉, **평균적인 응답 속도는 Between 방식에 비해 확실히 안정화**되었고,  
줌/팬이 자주 일어나는 실제 사용자 맵 인터랙션 환경에서는  
**체감 성능이 더 일정하고 예측 가능해졌다는 점이 장점**이다.


**공간 인덱스(PostGIS)를 쓰면 평균 응답뿐 아니라 분산이 줄어든다**
- ST_Intersects + GiST 인덱스를 쓰면서  
    **줌 인/아웃, 주변 탐색 시 전반적인 응답 시간이 안정화**되는 효과
- 일부 극단 케이스를 제외하면 대부분 20~80ms에 몰려 있음.

---
### 5.6.  정리 

기존에는 단순히 `latitude/longitude`에 `Between` 조건을 사용하는 방식으로 뷰포트 내 장소를 조회했는데, 좁은 범위에서는 빠르게 응답하지만 지도 이동 폭이 커지면 최대 900ms까지 지연이 튀는 문제가 있었다.

이를 개선하기 위해 `PostGIS`의 `ST_Intersects`와 `GiST` 공간 인덱스를 도입하여 뷰포트를 하나의 폴리곤(`ST_MakeEnvelope`)으로 보고, 해당 영역과 장소의 위치 컬럼(`location`)이 교차하는지를 기준으로 필터링하도록 변경했다.

그 결과, 대부분 케이스에서 응답 시간이 20~80ms 수준으로 안정화되었고, 실제 줌 인/아웃, 드래그 기반 탐색 시 체감 성능이 크게 개선되었다. 다만 극단적으로 넓은 범위를 한 번에 조회하는 경우에는 여전히 수백 ms 단위의 스파이크가 관측되어, 향후 다른 방법의 최적화를 생각해봐야 할 것 같다.(지금은 AI Agent 기능만으로 너무 바빠서 ㅠ)`

---
### 5.7.  번외 - 성능 분산 원리 
>[!QUESTION] 왜 GIST 인덱싱할 때 성능 분산이 줄었는가?


GiST 인덱스는 공간 데이터를 R-Tree 기반으로 계층적으로 분할해 필요한 영역만 탐색한다.
이로 인해, 조회 범위가 바뀌어도 매번 전체 테이블을 훑지 않고 검색 후보 집하을 안정적으로 제한한다.
![Pasted image 20251209122757.png](/img/user/supporter/image/Pasted%20image%2020251209122757.png)
- 2차원 좌표를 위의 사진과 같이 영역(노드)단위로 분할해 저장
- 필요한 박스만 검사 ➡후보 집합이 일관됨 


> `ST_Distance` 계산 자체가 빠른게 아니라, 계산해야 하는 대상이 적어짐 ➡ 지연 분산이 감소 

*핵심 정리*
1. **후보 집합이 안정적**
2. **2차원 공간을 인덱싱 처리 가능** 

*면접 답변*
- “Between 방식은 위도·경도를 각각 비교하는 단순 범위 필터라 조회 범위가 조금만 넓어져도 인덱스를 부분적으로만 사용하다가 금방 대량 스캔으로 전환됩니다. 또한 캐시나 테이블 분포에 따라 실행 계획도 자주 바뀌어 성능 편차가 큽니다.
- 반면 GiST 인덱스는 R-Tree 기반으로 공간을 계층적으로 분할해서, 조회 영역과 실제로 겹치는 노드만 탐색합니다. 즉, 전체 데이터 중 ‘후보 집합’을 매우 안정적으로 제한하고, 그 소수의 후보에만 정확한 거리 계산을 수행합니다.
- 그래서 조회 범위가 변해도 매번 전체 테이블을 뒤지지 않고, 인덱스가 안정적으로 pruning을 해주기 때문에 **평균 속도뿐 아니라 지연 분산까지 크게 줄어듭니다**



