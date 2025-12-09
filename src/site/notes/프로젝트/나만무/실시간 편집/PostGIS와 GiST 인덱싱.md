---
{"dg-publish":true,"permalink":"/프로젝트/나만무/실시간 편집/PostGIS와 GiST 인덱싱/","noteIcon":"","created":"2025-12-03T14:53:06.804+09:00","updated":"2025-12-09T17:20:11.338+09:00"}
---


> [!INFO] Preview
> - 만약 공간 검색, 거리 계산, 반경 조회 등을 할 거라면  
> - **PostGIS** 확장을 사용하는 게 훨씬 강력
> ```sql
> CREATE EXTENSION postgis;  
> CREATE TABLE places (   
> 		id SERIAL PRIMARY KEY,
> 		name VARCHAR(100),
> 		geom GEOGRAPHY(Point, 4326)  -- WGS84 좌표계 (위경도);
> ```



## 1.  개념 

> PostgreSQL에 "지도/좌표용 데이터 타입 + 연산자 + 인덱스"를 붙여주는 Extension

일반 DB의 숫자/문자열 중심 연산 수준을 넘어서, PostGIS는 "위치 기반 서비스"기능을 모두 DB 레벨에서 수행할 수 있게 한다
- 점-선-폴리곤 같은 **공간 데이터 타입**제공
- 실제 지구 곡률을 반영한 거리 계산 가능
- 반경 검색, 포함 여부 등 다양한 공간 연산 수행 가능
- GiST 인덱스를 통해 성능을 수십 ~ 수백 배 향상 시킬 수 있다.

PostGIS의 `geography` 타입은 지구 곡률을 반영한 거리 계산을 지원

Haversine과 동일한 수준의 정확도를 가지면서도 **인덱스를 활용**할 수 있다는 점이 가장 큰 장점

## 2.  주요 속성 

### 2.1.  geometry 타입

*개념*
- 좌표계를 가진 평명 기반 타입
- 대표적인 좌표계 = SRID 4326
- 좌표계(SRID)에 맞춰서 x, y 로 저장한다
- 단위 = degree 

*장점*
- 빠르다
- 기능 지원 폭이 넓다
- GiST 연산이 매우 다양하다

*단점 💢*
- 곡률을 고려하지 않는다 
  
### 2.2.  geography 타입
*개념*
- 지구 곡률을 반영한 타원체 기반 거리 계산을 수행하는 타입
- 기본 SRID 4326 위경도 좌표 사용
- 거리 단위 = meter => 반경 검색에 최적

*장점*
- Haversine과 동일한 수준의 **정확도** 
- **GiST 인덱스를 완벽히 지원**한다
- 미터 단위 반경 계산이 직관적이다.

*단점*
- geometry보다 연산이 살짝 무겁다 

### 2.3.  대표 함수들 

- `ST_MakePoint(lon, lat)` : 위경도 ➡ 점 생성
- `ST_SetSRID(geom, 4326)` : SRID 설정
- `ST_Distance(a, b)` : 두 점 사이의 거리
- `ST_DWithin(a, b, radius)` : a와 b가 radius 이내에 있는지 여부
- `ST_Contains / ST_Within / ST_Intersects` : 포함, 교차 여부


### 2.4.  geography vs geometry 정리

| 구분    | geometry    | geography     |
| ----- | ----------- | ------------- |
| 개념    | 평면 좌표       | 구면 좌표(지구 기반)  |
| 계산 단위 | degree(도)   | meter(미터)     |
| 곡률 반영 | X           | O             |
| 성능    | 빠름          | 약간 무거움        |
| 특징    | GIS 기능 풍부   | 반경 검색에 최적     |
| 주 사용처 | 지도 도형, 행정구역 | 반경 기반 LBS 서비스 |


반경 기반 검색은 대부분 `geography`가 적합하다.



## 3.  GiST 인덱스
Generalized Search Tree 

> 범용 인덱스 프레임워크 
> - 범위, 공간, 유사도, 최근접 검색 같은 복잡한 비교를 가능하게 해줌 



### 3.1.  일반 인덱스와 다른 점 

`B-Tree`
- 1차원 정렬 기반 : 값이 선형적으로 정렬됨 
- `WHERE` 절 같은 조건에 최적화되어 있다.
- 2D 데이터에는 부적합하다.

`GiST`
- 구조 자체가 "트리인데, 각 노드가 어떤 공간/범위를 대표하는 식"
	- 영역에 포함되는지 안되는지 이런식으로 검색(다차원 공간 검색)
- 각 노드가 Bounding Box를 가진다
- 이 사각형이 이 쿼리 영역과 겹치는지 여부만으로 **후보를 빠르게 줄임**<BR>![Pasted image 20251128235239.png](/img/user/supporter/image/Pasted%20image%2020251128235239.png)

### 3.2.  연산자

- `&&` : 두 geometry의 bounding box가 겹치냐?
- `<->` : 두 geometry의 거리(최근접 KNN 검색에서 사용)
- `@>` / `<@` : 포함/포함됨

## 4.  반경 검색 최적화 예시 
```sql
SELECT id, name
FROM places
WHERE ST_DWithin(
		location,
		ST_SetSRID(ST_MakePoint(:lng, :lat), 4326)::geography,
		:radius
)
ORDER BY location <-> ST_SetSRID(ST_MakePoint(:lng, :lat), 4326)::geography
LIMIT 50;
```



## 5.  스키마 설계 예시

아래와 같은 스키마 설정이 필요하다 
- 기존의 places 테이블에 GEOGRAPHY타입의 컬럼을 추가한다.
- GIST 인덱스를 적용한다
```sql
# ✅ GEOGRAPHY 타입 컬럼 추가 
ALTER TABLE places
    ADD COLUMN location GEOGRAPHY(POINT, 4326);

# ✅geometry 타입인 경우
## 기본적으로 평면 계산(곡률 무시)
UPDATE places
		SET location = ST_SetSRID(ST_MakePoint(lng, lat), 4326)
		
# ✅geography 타입
## 곡률을 고려한 거리 계산을 수행 (더 현실적)
## ST_SetSRID로 좌표계를 설정한 후 반드시 ::geography로 타입 캐스팅해야 함 
UPDATE places
		SET location = ST_SetSRID(ST_MakePoint(lng, lat), 4326)::geography		
    
# ✅인덱스 생성    
CREATE INDEX IF NOT EXISTS idx_places_location
    ON places USING GIST(location);    
```




## 6.  질문들 

아래는 예상 질문들 

### 6.1.  GIST 인덱스가 어떻게 반경 내 후보를 빠르게 찾는가❓

- GiST는 **공간 데이터를 "영역"으로 압축해 트리 구조로 저장**한다
- 반경 검색(`ST_DWithin`)을 실행하면 쿼리 반경이 만드는 원을 bounding box 영역으로 변환한다.
- 이 영역과 겹치는 노드만 탐색 
- 이를 통해 전체를 훑는게 아니라, 반경에 걸릴 가능성이 있는 영역만 빠르게 탐색해서 후보를 추리게 되어 실제 거리 계산은 아주 소수 


### 6.2.  B-tree 인덱스는 공간 연산에 왜 못 쓰는가❓
일반 B-tree 인덱스는 1차원 정렬을 위한 것이다.
위도/경도는 2차원이다.

