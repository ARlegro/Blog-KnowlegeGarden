---
{"dg-publish":true,"permalink":"/프로젝트/나만무/DB관련/PostGIS + GiST 기술적 챌린지 정리/","noteIcon":"","created":"2025-12-03T14:53:06.736+09:00","updated":"2025-12-09T17:20:11.175+09:00"}
---



## 1.  공간 인덱싱을 통한 성능 개선 

> - 요구 사항 : “현재 위치 기준 반경 N km 안의 장소를 찾는다”
> - 개선 결과 : 기존 방식 대비 **최대 181배 성능 향상** 수치

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


### 1.1.  Haversine 공식 사용 
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

### 1.2.  PostGIS 공간 인덱스 사용 
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




