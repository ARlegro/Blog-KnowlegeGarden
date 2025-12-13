---
{"dg-publish":true,"permalink":"/프로젝트/나만무/DB관련/PostGIS와 GiST 인덱싱/","noteIcon":"","created":"2025-12-03T14:53:06.804+09:00","updated":"2025-12-13T09:28:13.112+09:00"}
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


---
## 1.  PostGIS 개념 

> PostgreSQL에 "지도/좌표용 데이터 타입 + 연산자 + 인덱스"를 붙여주는 Extension

일반 DB의 숫자/문자열 중심 연산 수준을 넘어서, PostGIS는 "위치 기반 서비스"기능을 모두 DB 레벨에서 수행할 수 있게 한다
- 점-선-폴리곤 같은 **공간 데이터 타입**제공
- 실제 지구 곡률을 반영한 거리 계산 가능
- 반경 검색, 포함 여부 등 다양한 공간 연산 수행 가능
- GiST 인덱스를 통해 성능을 수십 ~ 수백 배 향상 시킬 수 있다.

PostGIS의 `geography` 타입은 지구 곡률을 반영한 거리 계산을 지원

Haversine과 동일한 수준의 정확도를 가지면서도 **인덱스를 활용**할 수 있다는 점이 가장 큰 장점


---
## 2.  주요 속성 

---
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

---
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

---
### 2.3.  대표 함수들 

- `ST_MakePoint(lon, lat)` : 위경도 ➡ 점 생성
- `ST_SetSRID(geom, 4326)` : SRID 설정
- `ST_Distance(a, b)` : 두 점 사이의 거리
- `ST_DWithin(a, b, radius)` : a와 b가 radius 이내에 있는지 여부
- `ST_Contains / ST_Within / ST_Intersects` : 포함, 교차 여부

---
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

---
## 3.  GiST 
Generalized Search Tree(확장 가능한 균형 트리)

#템플릿 

### 3.1.  개념 
>[!QUESTION] GIST란❓
> 특정 자료구조(ex. R-Tree)를 구현해주는 **프레임워크** <br>
> 범위, 공간, 유사도, 최근접 검색 같은 복잡한 비교를 가능하게 해줌 

❌GiST는 그 자체가 GK나의 고정된 자료구조가 아니다 <BR>
✅트리의 뼈대, 비교 연산 주입 인터페이스 제공 

GiST가 가능하게 하는 검색 유형
- 범위 검색 
- 공간 검색
- 유사도 검색
- 최근접 이웃 검색 

### 3.2.  구성 - 노드
#### 3.2.1.  전체 트리 구조 

GiST인덱스는 B-Tree와 유사하게 아래 3계층 구조를 가진다
1. 루트 노드 
2. 내부 노드
3. 리프 노드

But 노드가 의미하는 **Key의 해석방식이 B-Tree와 완전히 다르다**

#### 3.2.2.  노드 내부 구조 

모든 GiST 노드는 **하나의 페이지(Page = Block)** 로 구성되며, 그 안에는 여러 개의 엔트리가 배열 형태로 저장된다.
```cpp
struct GiSTEntry {
    Key key;          // ex. MBR(최소 경계 사각형)
    Pointer child;    // 자식 노드 or Heap TID
    bool compressed;  // Key 압축 여부
}
```
- *Key*
	- **하위 영역을 대표**하는 요약 정보
	- 공간 인덱스에서는 MBR
	- 내부 노드일수록 **하위 노드들의 MBR을 감싸는 더 큰 MBR ❗**
- *Pointer*
	- 어떤 노드인지에 따라 다름
	- 내부 노드 : 자식 GiST 페이지 
	- 리프 노드 : 실제 테이블 row를 가리키는 Heap TID

> 즉, 정확한 값은 리프에만 있고 상위 노드에는 대충 이 근처라는 요약본만 있음 



### 3.3.  PostGIS와 GiST의 관계

PostGIS의 공간 인덱스는
- **GiST 프레임워크 위에**
- **R-Tree 계열의 공간 분할 로직을 얹은 구현체**

즉, 
- GiST = 틀
- R-Tree = 공간 데이터를 다루는 규칙 집합
- PostGIS = 그 규칙을 geometry/geography 타입에 맞게 구현한 확장

### 3.4.  핵심 특징 요약 

- **B-Tree처럼 항상 균형 유지**
- **R-Tree처럼 MBR(Minimum Bounding Rectangle) 저장**
- 한 페이지(노드)에 **수십~수백 개의 MBR 엔트리** 저장
- 플러그 가능한 비교 함수 구조 (확장성)
- **데이터 압축** 
	- 고정 길이의 비트맵 바이트로 변환하여 저장
	- 장점 : 필요한 노드만 검색할 때 빠름
- **손실 인덱스(Lossy Index) 가능**
	- 저장되는 Key가 실제 데이터를 100% 표현하지 못할 수 있다
	- 단순화된 MBR로 저장화므로 더욱 손실적. 왜냐면 MBR은 사각형이라 **실제 다각형, 구불구불 곡선, 복잡 경로를 전부 반영할 수 없기** 때문이다.
	- 즉, Index레벨에서는 treu라도 실제 비교는 false일 수 있다.
	- 물론 postgreSQL은 recheck 기능으로 한번 더 확인함 

> 균형 유지로 인해 트리 depth가 짧다 + 하나의 노드에 수십~수백개의 MBR 저장 ➡ 디스크 I/O 횟수 줄음 

---
## 4.  GiST 인덱스

### 4.1.  개념 

> 복잡한 데이터 타입과 쿼리 패턴을 처리하기 위해 설계된 **고유연성·고확장성 인덱스 구조**


GiST가 가진 무기
1. ✅사용자 정의 비교
2. ✅다양한 데이터 타임
3. ✅공간, 유사도, 거리 기반 검색 


대신, GiST가 포기한 것 
1. ❌완전한 정렬 
2. ❌단일 비교 연산자 

### 4.2.  공간 인덱싱 생성 과정 

- 각 geometry 객체에 대해 MBR 계산
- 리프 노드에 MBR + Heap TID 저장
- 상위 노드로 올라가며 MBR을 병합

### 4.3.  GiST 인덱싱의 실제 동작 구조 (삽입)

### 4.4.  GiST 인덱싱의 실제 동작 구조 (검색)

1. **시작** : 루트 노드에서 검색 시작
2. **노드 탐색** : 현재 노드의 각 엔트리에대해 `consistent`함수를 호출
	- consistent함수 : 쿼리 조건과 노드의 범위를 비교 ➡검색 값이 해당 노드의 **하위 트리에 포함될 가능성이 있는 판단**  ex. 검색 영역이 노드의 MBR과 겹치는지 
	- 가능성이 있다면 : 해당 자식 노드로 내려감
	- 가능성이 없다면 : 해당 분기 무시됨
	  
3. 리프 노드 도달 


---
### 4.5.  일반 인덱스와 다른 점 

`B-Tree`
- 1차원 정렬 기반 : 값이 선형적으로 정렬됨 
- `WHERE` 절 같은 조건에 최적화되어 있다.
- 2D 데이터에는 부적합하다.

`GiST`
- 구조 자체가 "트리인데, 각 노드가 어떤 공간/범위를 대표하는 식"
	- 영역에 포함되는지 안되는지 이런식으로 검색(다차원 공간 검색)
- 각 노드가 Bounding Box를 가진다
- 이 사각형이 이 쿼리 영역과 겹치는지 여부만으로 **후보를 빠르게 줄임**<BR>![Pasted image 20251128235239.png](/img/user/supporter/image/Pasted%20image%2020251128235239.png)

---
### 4.6.  연산자

- `&&` : 두 geometry의 bounding box가 겹치냐?
- `<->` : 두 geometry의 거리(최근접 KNN 검색에서 사용)
- `@>` / `<@` : 포함/포함됨

---
## 5.  반경 검색 최적화 예시 
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


---
## 6.  스키마 설계 예시

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



---
## 7.  질문들 

아래는 예상 질문들 

### 7.1.  GiST

- GiST는 R-Tree 기반의 MBR 분할 구조를 가진 범용 인덱스 프레임워크로,  
- 공간 도형을 **최소 경계 사각형(MBR)으로 계층적으로 분할**해 나가기 때문에 **불필요한 I/O를 거의 제거**하고 **공간 검색 성능**을 극적으로 끌어올린다.

---
### 7.2.  GIST 인덱스가 어떻게 반경 내 후보를 빠르게 찾는가❓

- GiST는 **공간 데이터를 "영역"으로 압축해 트리 구조로 저장**한다
- 반경 검색(`ST_DWithin`)을 실행하면 쿼리 반경이 만드는 원을 bounding box 영역으로 변환한다.
- 이 영역과 겹치는 노드만 탐색 
- 이를 통해 전체를 훑는게 아니라, 반경에 걸릴 가능성이 있는 영역만 빠르게 탐색해서 후보를 추리게 되어 실제 거리 계산은 아주 소수 

---
### 7.3.  B-tree 인덱스는 공간 연산에 왜 못 쓰는가❓
일반 B-tree 인덱스는 1차원 정렬을 위한 것이다.
위도/경도는 2차원이다.


