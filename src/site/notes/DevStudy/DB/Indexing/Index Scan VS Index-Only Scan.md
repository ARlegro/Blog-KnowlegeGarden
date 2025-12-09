---
{"dg-publish":true,"permalink":"/DevStudy/DB/Indexing/Index Scan VS Index-Only Scan/","noteIcon":"","created":"2025-12-09T17:19:47.361+09:00","updated":"2025-12-09T17:19:47.363+09:00"}
---



## 1.  세팅 
- 테이블 명 : students
- 컬럼 : id, grade, name 
- 데이터 수 : 1천만명 
```SQL
CREATE TABLE students
(
    id INTEGER GENERATED ALWAYS AS IDENTITY,
    name VARCHAR(10),
    grade INTEGER CHECK (grade >= 0 AND grade <= 100)
)
```
```SQL 
INSERT INTO students(grade, name)
SELECT 
    Floor(random() * 100),
    '학생' || LPAD(n::text, 8, '0')
FROM generate_series(1, 10000000) AS s(n)
```


## 2.  기본 조회 분석
> 데이터베이스가 이 행을 가져오기 위해 무엇을 했는지 파악하는데 집중하기 

### 2.1.  전체 조회 
```SQL 
EXPLAIN ANALYZE SELECT * FROM students

QUERY PLAN

Seq Scan on students  (cost=0.00..212501.70 rows=13897170 width=15) (actual time=7.018..1725.139 rows=10000000 loops=1)
Planning Time: 0.382 ms
JIT:
  Functions: 2
  Options: Inlining false, Optimization false, Expressions true, Deforming true
  Timing: Generation 0.231 ms (Deform 0.113 ms), Inlining 0.000 ms, Optimization 0.496 ms, Emission 6.378 ms, Total 7.105 ms
Execution Time: 2078.589 ms
```
- 실행 시간 : 2078.589ms (매우 느리다)


### 2.2.  ID 조건 값 조회 
```SQL 
EXPLAIN ANALYZE SELECT * FROM students WHERE id = 123
```

```SQL
QUERY PLAN

Gather  (cost=1000.00..126613.85 rows=1 width=23) (actual time=366.689..374.091 rows=0 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on students  (cost=0.00..125613.75 rows=1 width=23) (actual time=337.189..337.189 rows=0 loops=3)
        Filter: (id = 123)
        Rows Removed by Filter: 3333333
Planning Time: 0.812 ms
JIT:
  Functions: 12
  Options: Inlining false, Optimization false, Expressions true, Deforming true
  Timing: Generation 1.610 ms (Deform 0.625 ms), Inlining 0.000 ms, Optimization 1.387 ms, Emission 25.613 ms, Total 28.610 ms
  
Execution Time: 422.612 ms
```
- 조금 더 빨라지긴 했는데 여전히 많이 느리다.
- **느린 이유** 
	- `ID = 123`이라는 조건에 인덱싱이 되어 있지 않기 때문에, 데이터베이스는 자신이 아는 방법인 **병렬 순차 스캔(Parallel Sequence Scan)** 을 수행하기로 결정
	- 이는 Table Scan이며 테이블 자체로 이동하여 말 그대로 한 줄씩 검색하고, 각 행을 가져온 다음`ID`가 7인지 확인(ID = 7 행을 찾을때까지)


## 3.  Index Scan
### 3.1.  ID 인덱스 생성 후 전체 컬럼 조회 
> B-트리 인덱스를 생성하여 검색 속도를 조금 더 빠르게 만들 것
```SQL 
CREATE INDEX id_index ON students(id)
```

#### 3.1.1.  WHERE 조건 있을 때 똑같이 실행 
```SQL 
SELECT * FROM grades WHERE ID = 7

Index Scan using id_index on students  (cost=0.43..8.45 rows=1 width=23) (actual time=0.031..0.031 rows=0 loops=1)
  Index Cond: (id = 123)
Planning Time: 0.634 ms
Execution Time: 0.089 ms
```
- 매우 매우 빨라졌다. (422.6 ➡ 0.09)
- 이것은 Index Scan이다 Cuz `ID_index`라는 인덱스를 사용하여 `ID = 7`을 검색했기 때문

#### 3.1.2.  WHERE 조건 없을 때 똑같이 실행 
> [!WARNING] WHERE 절이 없으면 전체를 조회하기에 여전히 느리다
```SQL
EXPLAIN ANALYZE SELECT id, name FROM students
```

```SQL 
QUERY PLAN

QUERY PLAN

Seq Scan on students  (cost=0.00..163696.15 rows=10000115 width=23) (actual time=0.089..1024.483 rows=10000000 loops=1)
Planning Time: 0.392 ms
Execution Time: 1478.668 ms
```
- Indexing을 생성했음에도 실행시간이 굉장히 느리다.
- 이것은 아직도 여전히 힙영영의 Table Scan이 필요하기 때문 

### 3.2.  Index Scan의 실제 동작 

#### 3.2.1.  흐름 
인덱스에서 각 인덱스 값은 page 번호와 Row ID를 갖고 있다.
우선 10개의 인덱스 값이 있다고 가정하자.
1. 인덱스 탐색 (B-Tree 가정)
	- 인덱스 탐색으로 각 값들의 CTID(page번호, row 오프셋)를 읽는다.
		```TEXT
		 키 값  …  CTID
		  A       (page 217, off 5)
		  B       (page  42, off 9)
		  C       (page 998, off 2)
		  D       (page  42, off 11)
		```
2. 힙 페이지 접근 
	- CTIP가 가리키는 페이지에 접근하여 그 페이지 자체를 읽어온다.
	- 읽어온 페이지는 Shared Buffer에 저장 
3. 반복
	- 만약 다른 인덱스값이 2번을 수행하려하기 전에 Shared buffer에 해당하는 페이지가 있으면 추가 I/O 없이 Shared buffer에서 바로 행만 꺼낸다.
	- 이를 통해 불필요한 page 접근을 막음                                                                                           

#### 3.2.2.  Q. 왜 랜덤 I/O ❓
>[!QUESTION] 페이지를 중복해서 읽지 않는데 왜 랜덤 I/O가 발생한다고 할까 ❓
>- 페이지를 **중복해서** 읽지 않는 건 맞지만, **읽게 되는 ‘첫 페이지들’ 자체가 디스크 여기저기에 흩어져** 있다
>- 디스크 헤드가 **매번 위치를 바꿔야** 하므로 여전히 랜덤 I/O

가령 아래와 같은 인덱스 키값과 CTID가 있다고 가정하자 
```TEXT 
키 값  …  CTID
	A       (page 217, off 5)
	B       (page  42, off 9)
	C       (page 998, off 2)
	D       (page  42, off 11)
	…
```
- 키는 ABCD… 식으로 순서가 잡히지만 **page 217 → 42 → 998 → 42 …** 처럼 **페이지 번호가 튀는** 모습
- 애초에 DB의 테이블은 인덱스 키 값을 기준으로 정렬된게 아니므로 이 상황이 당연한거다.
- 따라서 인덱스 순서대로 페이지에 접근할 때, 디스크 헤드는 **페이지마다 점프**해야 하는 문제가 발생한다.

> 즉, "Index Scan"은 **CTID 순서가 물리 페이지 순서와 무관**해서 처음 읽는 페이지마다 **디스크·OS가 위치를 바꿔야** 하고, 이것이 곧 **랜덤 I/O 비용**
> 
>   > 반면, Seq Scan은 헤드가 연속 트랙을 따라가서 블록을 한꺼번에 읽는 장점이 있다.


**해결 방법** 
1. **클러스터링** 
	- `CLUSTER grades USING grades_name_idx;`
	- 근데 PostgreSQL에서 클러스링은 일회성이라고 들음 
2. **카디널리티 낮은 조건**으로 해서 Random I/O 발생 확률 줄이기 
3. **Bitmap Scan**
	- **먼저 Page ID를 모아 정렬 후 순차 읽기**로 이 랜덤 I/O를 줄인다.


#### 3.2.3.  면접 답변 
index Scan은 **키 순서대로 CTID**를 뽑는데, CTID 안의 **페이지 번호는 디스크 여기저기에 흩어져** 있다.  
중복 페이지는 한 번만 읽긴 하지만, **새 페이지에 닿을 때마다 디스크 헤드가 점프**하므로 여전히 **랜덤 I/O** 특성을 갖는다.  
랜덤 I/O를 해결하는 방법으로는 Bitmap Scan이 있습니다. 이 방법은 **페이지 번호를 먼저 모아 정렬**한 뒤 순차로 읽어 랜덤 I/O을 크게 줄여줍니다
이 외에도 카디널리티가 낮은 조건으로 SQL문을 작성하거나 클러스터링을 통해 인덱스 값을 기준으로 힙페이지를 정렬하는 방법 또한 있습니다.


## 4.  Index-Only-Scan
### 4.1.  Index-Only-Scan의 예시 
> ID 인덱스 생성 후 ID 컬럼만 조회 

예전에 썼던 글에 다 나왔던 내용

```SQL
EXPLAIN ANALYZE SELECT ID FROM students WHERE id = 123
```

```SQL 
QUERY PLAN

Index Only Scan using id_index on students  (cost=0.43..4.45 rows=1 width=4) (actual time=0.055..0.056 rows=0 loops=1)
  Index Cond: (id = 123)
  Heap Fetches: 0
Planning Time: 0.568 ms
Execution Time: 0.074 ms
```
- Index-Only-Scan을 사용한다. 
- 데이터베이스가 테이블로 이동하지 않아도 된다. 테이블로 갈 필요가 없었기 때문에 인덱스 온리 스캔. 
- 만약 이게 가능하다면 정말 빠를 것 

> [!missing]  But 상식적으로 ID를 찾으려고 ID를 기준으로 `SELECT`한다는 것은 비상식적이다.

>[!SUCCESS]  실제 상식적으로 실용적인 Index-Only-Scan 방법을 어떻게 할 지 알아볼 것 



### 4.2.  실용적 Index-Only-Scan (feat. 커버링 인덱스)
> Non-Key Columns 에 주목 

> [!TIP] 클라이언트가 ID를 기준으로 name을 자주 요청한다는 가정 

>[!tip] 인덱스 온리 스캔(Index Only Scan) 조건
>1. 커버링 인덱스로 필요한 모든 컬럼이 인덱스에 포함 
>2. **visibility map** 상에 “이미 죽지 않은(visible) 튜플”임이 표시돼 있어야 함 (`VACUUM TABLE students`)
>3. **정렬**(`ORDER BY id`)이나 **필터**(`WHERE id = …`)가 있어야 옵티마이저가 인덱스를 타고 탐색할 동기가 생김
>	- 이게 없으면 걍 Seq Scan이 더 빠름 



#### 4.2.1.  세팅 1. include 인덱스 생성 
기존 Index 삭제 후 재 생성 
```SQL
DROP INDEX id_index 

CREATE INDEX id_idx ON grade(id) INCLUDE (name)
```
INCLUDE를 포함한 컬럼을 'Non-Key Column'이라고 한다.
위 예시에서:
- id : key column
- name : non-key column

#### 4.2.2.  세팅 2. VACCUME ⭐ - **visibility map** 갱신 
- PostgreSQL은 MVCC 구조이기 때문에, **해당 테이블의 튜플(행)** 이 **인덱스만으로 실제 테이블에 접근하지 않아도 되는 상태인지**를 확인해야 한다.
- 이건 내부적으로 "visibility map"을 통해 확인된다.
- 그런데 **VACUUM** 또는 **autovacuum**이 제대로 돌지 않았거나,
- 최근에 **대량 삽입/수정/삭제**가 이루어졌다면, visibility map이 갱신되지 않아 Index Only Scan이 불가능할 수 있다. 
- 따라서 아래와 같이 필요 
```SQL
VACUUM ANALYZE students;
```

>[!tip] Visibility Map
>- PostgreSQL이 각 테이블 블록(페이지)의 **모든 튜플(행)이 "모두 visible"한지** 여부를 추적하는 **비트맵**
>- 역할 **Index Only Scan 시 테이블을 굳이 안 봐도 돼"** 를 알려주는 지도 역할

>[!tip] VACUUM ❓
>- PostgreSQL이 테이블의 **죽은 튜플(dead tuples)**을 정리하고, **visibility map을 갱신**하는 명령
>- `Index Only Scan`을 쓸 수 있는 조건 중 하나가 **visibility map이 최신 상태**이기 때문


#### 4.2.3.  실행 (커버링 인덱스가 효과 ❌)

```SQL
                               Table "public.students"
 Column |         Type          | Collation | Nullable |           Default            
--------+-----------------------+-----------+----------+------------------------------
 id     | integer               |           | not null | generated always as identity
 grade  | integer               |           | not null | 
 name   | character varying(20) |           |          | 
Indexes:
    "id_index" btree (id) INCLUDE (name)
```

```SQL
EXPLAIN ANALYZE SELECT name FROM students WHERE id = 8

Index Only Scan using id_index on students  (cost=0.43..4.45 rows=1 width=15) (actual time=0.136..0.136 rows=0 loops=1)
  Index Cond: (id = 8)
  Heap Fetches: 0
Planning Time: 0.510 ms
Execution Time: 0.183 ms
```
- Index Only Scan

```SQL 
VACUUM ANALYZE students; << 일단 이거하고 

EXPLAIN ANALYZE SELECT id, name FROM students

QUERY PLAN

Seq Scan on students  (cost=0.00..173528.84 rows=9999884 width=19) (actual time=4.769..1120.245 rows=10000000 loops=1)
Planning Time: 0.418 ms
JIT:
  Functions: 2
  Options: Inlining false, Optimization false, Expressions true, Deforming true
  Timing: Generation 0.220 ms (Deform 0.146 ms), Inlining 0.000 ms, Optimization 0.535 ms, Emission 4.078 ms, Total 4.833 ms
Execution Time: 1458.141 ms
```
**❓id, name이 전부 Index에 있는데 Seq Scan이 발생한 이유 ❓**
- Index Only Scan이 _가능_하더라도 플래너는 **비용**이 낮은 경로를 고른다.
- `WHERE` 없이 10 M 행 전체를 읽는 쿼리는 **연속적인 Seq Scan이 더 저렴**하다고 판단되기 때문에 Index Only Scan을 선택하지 않는 것이 정상 동작
- 커버링 인덱스가 있어도 조건 없이 전 행을 읽으면 **인덱스 블록 × 랜덤 I/O**가 **테이블 × 순차 I/O**보다 비싸게 계산돼서, PostgreSQL은 Seq Scan을 고른다.

>핵심 : 인덱스를 모두 읽는 작업이 생각보다 큰 비용


### 4.3.  INCLUDE가 빛을 발하는 상황 

#### 4.3.1.  정렬 조건이 있을 때 (ORDER BY)

```SQL
EXPLAIN ANALYZE SELECT id, name FROM students ORDER BY id

QUERY PLAN

Index Only Scan using students_id_name_idx on students  (cost=0.43..347750.50 rows=9999871 width=18) (actual time=0.030..977.727 rows=10000000 loops=1)
  Heap Fetches: 0
Planning Time: 0.178 ms
Execution Time: 1201.986 ms
```
- **인덱스 ❌ :**
	- 시간 : 2827.419 ms
	- Flow : Paraller Seq Scan ➡ Sort ➡ Merge Sort 
	  
- **인덱스 ✅**
	- 시간 : 1201.986 (2배 차이) 
	- Flow : Index Only Scan 


**⭐참고로 이 효과는 LIMIT 과 있을 때 효과 만점 (다른 곳에서도 언급했던 내용)**
```SQL
EXPLAIN ANALYZE SELECT id, name FROM students ORDER BY id LIMIT 10000;

QUERY PLAN

Limit  (cost=0.43..348.19 rows=10000 width=18) (actual time=0.042..1.368 rows=10000 loops=1)
  ->  Index Only Scan using students_id_name_idx on students  (cost=0.43..347752.43 rows=10000000 width=18) (actual time=0.040..0.930 rows=10000 loops=1)
        Heap Fetches: 0
Planning Time: 0.236 ms
Execution Time: 1.593 ms
```
- 인덱스 ❌ : 1149 MS 
- 인덱스 ✅ : 1.593 ⭐⭐⭐
  
> 약 722배정도나 차이난다.


#### 4.3.2.  필터링 조건 있을 때 (WHERE)

```SQL
EXPLAIN ANALYZE SELECT id, name FROM students
WHERE id >= 5_100_000
LIMIT 1000;


QUERY PLAN

Limit  (cost=0.43..37.71 rows=1000 width=18) (actual time=0.092..0.249 rows=1000 loops=1)
  ->  Index Only Scan using students_id_name_idx on students  (cost=0.43..183078.97 rows=4911459 width=18) (actual time=0.091..0.196 rows=1000 loops=1)
        Index Cond: (id >= 5100000)
        Heap Fetches: 0
Planning Time: 0.054 ms
Execution Time: 0.284 ms
```
- 전체 데이터 10M
- 5M까지는 Seq Scan을 타는데 
- 필터링 row가 50퍼 이하로 떨어지니까 index-only-scan 타네
- 인덱스 ❌ : 5.272 MS 
- 인덱스 ✅ : 0.284 ⭐⭐⭐


> 계속 말하는거지만 전체를 조회할 때는 Seq Scan이 굉장히 효율적이다.


> [!WARNING] 인덱스가 무조건 좋은것은 아니다
> - 인덱스를 위한 heap fetch가 필요
> - 메모리 필요

