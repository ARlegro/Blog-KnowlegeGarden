---
{"dg-publish":true,"permalink":"/DevStudy/DB/Indexing/Partial Index in Postgres/","noteIcon":"","created":"2025-11-27T16:54:59.719+09:00","updated":"2025-12-08T17:28:02.208+09:00"}
---


Partial Index는 Postgres에서 제공하는 강력한 기능들 중 하나이다.


### 0.1.  조건부 인덱스 개념
> 테이블 전체의 인덱스를 거는게 아니라, **특정 조건을 만족하는 행들만 indexing하는 기능** 
- 일부 데이터만 인덱싱할 수 있어서 **인덱스가 불필요하게 커지는 것을 방지**할 수 있다.
  
```sql
CREATE INDEX idx_active_users 
ON users (email) 
WHERE is_active = true;
```

### 0.2.  언제 사용되는가❓

1. *가끔 검색하는 조건일 경우* 
	- 예를 들어, deleted = false인 경우만 가끔 검색하는 경우 
	- 만약 자주 검색한다면 그냥 index 거는게 좋음 
	  
2. 쿼리 패턴이 특정 조건에 집중될 때 


### 0.3.  심화 : partial unique index 

> 특정 조건을 만족하는 행에서 unique를 강제하는 index 

```sql
CREATE UNIQUE INDEX idx_unique_planday_title
ON plan_day (workspace_id, title)
WHERE deleted_at IS NULL;
```
- 일반 UNIQUE와 달리 조건 만족하는 행만 UNIQUE조건을 거는 형태 
- 매우 유연하고 실용적이다



### 0.4.  심화 : COPNCURRENTYL 옵션 
```SQL
CREATE INDEX CONCURRENTLY idx_place_user_review_has_content
    ON place_user_review (created_at)
    WHERE content IS NOT NULL;
```

> [!INFO] CREAT INDEX vs CREATE INDEX CONCURRENTLY 
> 1. `CREATE INDEX`
> 	- 테이블에 Write Lock이 걸림.
> 	- 이 인덱스를 생성하면서 write 트래픽은 전부 대기상태가 됨 
> 2. `CREATE CONCURRENTLY INDEX`
> 	- 테이블에 Wirte Lock ❌
> 	- ✅서비스가 계속 돌아가도 인덱스 생성가능
> 	- 💢But, 더 오래 걸리고, 트랜잭션 내부에서는 사용 불가 
> 	- 보통 실서비스를 운영중이고 큰 테이블에 인덱스를 추가할 때 사용 


