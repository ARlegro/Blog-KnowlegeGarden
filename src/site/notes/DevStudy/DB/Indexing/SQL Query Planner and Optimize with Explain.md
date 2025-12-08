---
{"dg-publish":true,"permalink":"/DevStudy/DB/Indexing/SQL Query Planner and Optimize with Explain/","noteIcon":"","created":"2025-06-08T21:09:37.231+09:00","updated":"2025-12-08T17:28:35.839+09:00"}
---

> PostgreSQL 관점 




>[!QUESTION] Postgre는 무슨 Query Plans을 짤까❓


EXPLAIN 다음에 쿼리문을 작성하면 Query Plan이 나온다

```SQL 
postgres=# EXPLAIN select * from employees;
                        QUERY PLAN                         
-----------------------------------------------------------
 Seq Scan on employees  (cost=0.00..16370.00 rows=1000000 width=18)
(1 row)

--------------------------------------------------
 Seq Scan on employees  (cost=0.00..16370.00 rows=1000000 width=18) (actual time=0.070..50.091 rows=1000000 loops=1)
 Planning Time: 0.059 ms
 Execution Time: 74.229 ms
(3 rows)
```

Seq Scan
- 순차적 스캔 = Full-Table-Scan이랑 거의 같다 
- 굉장히 비효율적 쿼리 Cuz 힙에 접근하고 모든 테이블에 접근하려하기 때문 

cost=0.00 .. 16370.00 
- PostgreSQL이 예측한 상대 비용
- 두개의 점 (..)을 기준으로 나뉜다.
- **첫 분할 숫자(0.00) 뜻**
	- 첫 페이지를 가져오는데 걸린 시간 (ms)
	- 0인 이유 : Postgres는 즉시 테이블에가서 첫 로우를 가져와서 결과를 반환?
	- 그래서 첫 결과의 비용은 거의 없는 것 
	- 이게 0이 아닐 경우 : 데이터를 가져오기 전에 몇가지 작업을 하는 것 
	  
- **두번째 분할 숫자 뜻** 
	- 총 걸리는 예상 시간 

rows 
- Planner가 예측한 결과 rows 수 
- 실제 수는 아니다 

width 
- 각 row의 평균 바이트 크기 
- 디스크 I/O, 메모리 사용량 계산 시 중요 기준이 됨
- 이거를 줄여야 network비용을 줄일 수 있음 ⭐
	- 이래서 `TEXT` 말고 `VARCHAR(10)`이런거 쓰나 봄 


ROW의 개수를 대략 파악하기에는 `EXPLAIN`이 좋다. 
- `SELECT COUNT`함수는 성능을 매우 저하시킨다
- 그래서 `EXPLAIN`으로 대량 얼마나 row있는지 확인 


## 1.  ORDER BY의 영향 실험 

#### 1.1.1.  세팅 
Grade 테이블에 2천만개의 데이터 존재 
```SQL 
INSERT INTO grades(name)
SELECT 
    LPAD ((floor(random() * 10000)::int)::text, 5, '0')
FROM generate_series(1, 20000000) 
```

인덱스 키 없는 테이블을 전체 조회한다했을 때 planner는비용이 많이 들거라고 예상했다.

#### 1.1.2.  기본 EXPLAIN
```SQL
EXPLAIN ANALYZE SELECT * FROM grades 

QUERY PLAN

Seq Scan on grades  (cost=0.00..288496.96 rows=20000096 width=6) (actual time=0.011..1184.254 rows=20000000 loops=1)
Planning Time: 0.152 ms
Execution Time: 1796.071 ms
```
2000만개의 데이터임에도 불구하고 Planner의 예상 rows는 다르다.
이제는 `ORDER BY` 문을 사용했을 때를 알아보자 


### 1.2.  ORDER BY가 미치는 영향 ⭐⭐
#### 1.2.1.  기본 ORDER BY (인덱스 X)
```SQL 
EXPLAIN SELECT * FROM grades ORDER BY name;

  
QUERY PLAN

Gather Merge  (cost=1358641.53..3303231.00 rows=16666746 width=6) (actual time=13728.863..17861.863 rows=20000000 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Sort  (cost=1357641.51..1378474.94 rows=8333373 width=6) (actual time=13658.816..14556.191 rows=6666667 loops=3)
        Sort Key: name
        Sort Method: external merge  Disk: 66776kB
        Worker 0:  Sort Method: external merge  Disk: 64088kB
        Worker 1:  Sort Method: external merge  Disk: 65064kB
        ->  Parallel Seq Scan on grades  (cost=0.00..171829.73 rows=8333373 width=6) (actual time=0.022..537.267 rows=6666667 loops=3)

Planning Time: 0.182 ms
Execution Time: 18573.938 ms
```
> 순서 : Parallel Seq Scan ➡ Sort ➡ Merge Sort

비용만 봐도 엄청 오래 걸릴 것으로 Planner가 예상했다.

Sort + Gather Merge
- Gather Merge ➡ 정렬 유지 상태로 데이터를 병합 (병합 정렬)
- Sort ➡ 병렬로 처리되는 worker들이 각자 맡은 부분의 데이터를 가져와서 데이터를 정렬한다.
- Worker들이 정렬한 데이터들을 병합정렬 시킨다. 

> 흐름 정리 : 워커별 스캔 ➡ 워커별 스캔데이터 정렬 ➡ 전역 병합 정렬 

**비싼 이유 분석** 
- `ORDER BY name` 때문에 파이프라인에 **Sort + Gather Merge**가 삽입됐고, 이것이 비용을 급격히 높였다.


**❓ORDER BY가 플랜을 어떻게 바꿨나**
일단, Planner는 인덱스가 없다면 순차 스캔 후 전역 병합 정렬이 가장 저렴하다고 판단했다.
그래서 아래와 같이 예상 플랜을 바꾼 것 

| 조건                                 | 예상 플랜                                                            |
| ---------------------------------- | ---------------------------------------------------------------- |
| **정렬 없음** (`SELECT * FROM grades`) | `Gather → Parallel Seq Scan` (오직 테이블 스캔 후 스트림 병합)                |
| **정렬 있음** (`ORDER BY name`)        | `Gather Merge → Sort → Parallel Seq Scan` (스캔한 뒤 워커별 정렬 + 전역 병합) |

>[!tip] 빈번히 같은 정렬을 요구한다면, **B-tree 인덱스 구축**이 가장 손쉬운 개선

#### 1.2.2.  인덱스 추가 후 ORDER BY 

```SQL 
CREATE INDEX ON grades(name);

EXPLAIN ANALYZE SELECT * FROM grades ORDER BY name;


QUERY PLAN

Index Only Scan using grades_name_idx on grades  (cost=0.44..369965.88 rows=20000096 width=6) (actual time=0.026..1401.230 rows=20000000 loops=1)
  Heap Fetches: 0
Planning Time: 0.158 ms
Execution Time: 2017.761 ms
```
무려 9배나 차이가 난다.
이번에는 Index Only Scan 

LIMIT을 안 썼는데도 이정도면 LIMIT 쓰면 엄청난 차이 발생 예상 가능 

#### 1.2.3.  인덱스(X, O) + ORDER BY + LIMIT
>미리 말하지만, 이 조합에서 인덱스가 있고 없고는 1000배의 성능까지도 차이가 날 수 있다.

**💢인덱스 X + LIMIT(10만)**
```SQL
EXPLAIN ANALYZE SELECT * FROM grades ORDER BY name LIMIT 100_000

QUERY PLAN

Limit  (cost=906568.27..918235.75 rows=100000 width=6) (actual time=16363.194..16415.650 rows=100000 loops=1)
  ->  Gather Merge  (cost=906568.27..2851157.73 rows=16666746 width=6) (actual time=16263.220..16309.685 rows=100000 loops=1)
        Workers Planned: 2
        Workers Launched: 2
        ->  Sort  (cost=905568.25..926401.68 rows=8333373 width=6) (actual time=16169.230..16173.659 rows=34510 loops=3)
              Sort Key: name
              Sort Method: external merge  Disk: 64048kB
              Worker 0:  Sort Method: external merge  Disk: 66808kB
              Worker 1:  Sort Method: external merge  Disk: 65064kB
              ->  Parallel Seq Scan on grades  (cost=0.00..171829.73 rows=8333373 width=6) (actual time=0.065..551.865 rows=6666667 loops=3)
Planning Time: 0.558 ms
JIT:
  Functions: 1
  Options: Inlining true, Optimization true, Expressions true, Deforming true
  Timing: Generation 0.110 ms (Deform 0.000 ms), Inlining 90.590 ms, Optimization 2.417 ms, Emission 6.947 ms, Total 100.064 ms
Execution Time: 16477.790 ms
```
- 어찌됐던 힙에 접근해서 병렬 순차 스캔 ➡ 정렬 ➡ 병합 정렬 Flow를 가진다.

**💚인덱스 O + LIMIT(10만)**
```SQL
CREATE INDEX ON grades(name);

EXPLAIN ANALYZE SELECT * FROM grades ORDER BY name LIMIT 100_000

QUERY PLAN

Limit  (cost=0.44..1850.26 rows=100000 width=6) (actual time=0.169..14.574 rows=100000 loops=1)
  ->  Index Only Scan using grades_name_idx on grades  (cost=0.44..369965.88 rows=20000096 width=6) (actual time=0.167..8.743 rows=100000 loops=1)
        Heap Fetches: 0
Planning Time: 0.201 ms
Execution Time: 17.698 ms
```
>[!tip] 1000배 달하는 차이 
>- 와우!!! 엄청난 차이가 발생 
>- 16477.8 ms ➡ 17.7 ms
- 현재 인덱스는 name 컬럼을 기준으로 만들어졌다.
- 기본적으로 **인덱스는 정렬된 상태**로 되어있기에 **ORDER BY 문과 궁합**이 잘 맞는다.
- 또한 현재는 오직 name필드만 가져오면 되기 때문에 Heap에 접근하지 않아도 된다.(**힙 fetches = 0**) 그렇기에 10만 데이터라도 매우매우 빠른 것 
- 핵심은 **Index Only Scan**이다
	- `Index Only Scan`은 **인덱스와 Visibility Map만 보고** 힙을 건너뛸 수 있다.

### 1.3.  추가 실험 
```SQL 
EXPLAIN SELECT * FROM grades where id = 10;

QUERY PLAN

Index Scan using grades_pkey on grades  (cost=0.44..8.46 rows=1 width=10)
  Index Cond: (id = 10)
```

