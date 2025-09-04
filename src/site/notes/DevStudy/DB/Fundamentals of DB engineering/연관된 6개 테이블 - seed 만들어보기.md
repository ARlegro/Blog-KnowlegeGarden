---
{"dg-publish":true,"permalink":"/DevStudy/DB/Fundamentals of DB engineering/연관된 6개 테이블 - seed 만들어보기/","noteIcon":"","created":"2025-06-12T13:06:38.479+09:00","updated":"2025-07-13T21:41:54.202+09:00"}
---


---
단일 테이블이라면 아래처럼 간단히 넣을 수 있겠지만, 실무는 그렇지 않을 것이다.
```SQL
INSERT INTO grades(name, g, firstname, lastname)
SELECT 
    '학생' || LPAD(Floor(random()*n)::text, 7, '0'),
    Floor(random()*100),
    '김이박',
    'Real Name' || LPAD(n::text, 7, '0')
    
FROM generate_series(1, 1_000_000) AS s(n)
```

---
그렇다면, 테이블끼리 다대다, 일대다, 일대일 등의 연관관계가 있다면?? 여러개라면??? 
이전에 내가 했던 프로젝트의 ERD를 보고 만들어보자 
아래 사진은 이전 프로젝트에서 했던 ERD의 일부이다.
![Pasted image 20250610133621.png](/img/user/supporter/image/Pasted%20image%2020250610133621.png)
- 알림 - 사용자(多 : 1)
- 사용자 - 구독(1 : 多)  *구독 = users_interests 로 다대다 중간 테이블이다*
- 관심사 - 구독(1 : 多)
- 관심사 - 기사_관심사(1 : 多)
- 기사 - 기사_관심사 (1 : 多)


---
### Step 1. 독립된 테이블 먼저 시드를 만든다.

독립된 테이블이란 다른 테이블을 참조하고 있지 않는 것이다.

위의 예에서 독립된 테이블이 뭘까???
- 사용자
- 관심사
- 기사 


---
### Step 2. 연관된 테이블 시도 

이 테이블들의 시드를 만들 때는 몇 가지 핵심 기법들이 있다.

---
#### 1. CROSS JOIN + 확률 조건

```sql
-- 모든 사용자 x 모든 관심사 조합에서 30% 확률로 연결
FROM users u 
CROSS JOIN LATERAL (
		SELECT id 
		FROM interests 
		ORDER BY random() 
		LIMIT floor(random() * 3 + 1)::int -- 1~3개 랜덤 선택 
) i
ON CONFLICT (user_id, interest_id) DO NOTHING;
```

✅`CROSS JOIN`
-  한쪽 테이블의 행 하나당 다른 쪽 테이블의 모든 행을 하나씩 모든 행들을 각각 조인
	![Pasted image 20250610140153.png](/img/user/supporter/image/Pasted%20image%2020250610140153.png)

✅`LATERAL`
- 외부(왼쪽) 테이블의 **각 행마다** 서브쿼리를 **새로 실행**”해서, 그 행별 값(`u.id` 등)을 서브쿼리 내부에서 **바로 참조**할 수 있다.
```SQL
-- LATERAL 없는 경우: 서브쿼리(인덱스 1~3)를 단 한 번만 뽑음
FROM users u
CROSS JOIN (
  SELECT id FROM interests ORDER BY random() LIMIT 3
) AS i

-- LATERAL 있는 경우: users의 각 행마다 서브쿼리를 다시 실행
FROM users u
CROSS JOIN LATERAL (
  SELECT id FROM interests ORDER BY random() LIMIT 3
) AS i
```
- **Without LATERAL**:
    - `interests`에서 무작위로 3개를 **한 번만** 뽑아 놓고,
    - 그 3개의 조합을 모든 `users` 행과 매핑합니다.
        
- **With LATERAL**:
    - `users`가 10,000명이라면,
    - `interests ORDER BY random() LIMIT 3`가 10,000번 실행되어
    - “사용자별 서로 다른 3개”를 보장합니다.

- i = 서브쿼리에 붙인 별칭 
- 이거 붙여야 `ERROR`가 안나네 

---
#### 2. 중복 방지
```sql
-- UNIQUE 제약조건이 있는 경우
ON CONFLICT (user_id, interest_id) DO NOTHING
```

---
#### 3. 실제 ID 참조

```sql
-- 하드코딩 대신 실제 테이블에서 ID 가져오기
SELECT u.id, i.id FROM users u, interests i
```

---
#### 4. 데이터 양 조절

```sql
-- LIMIT로 너무 많은 데이터 생성 방지
LIMIT 100000
```

---
#### ⚡ 실행 팁
1. **트랜잭션 사용** 권장:
```sql
BEGIN;
-- 시드 데이터 생성
COMMIT;
```
2. **배치 크기 조절**: 데이터가 너무 많으면 여러 번에 나누어 실행
3. **인덱스 고려**: 대용량 데이터 생성 전 인덱스를 임시로 제거했다가 나중에 다시 생성하면 더 빠름

이렇게 하면 현실적인 연관관계를 가진 테스트 데이터를 효율적으로 만들 수 있어요!




**LATERAL**은 각 사용자마다 별도로 관심사를 선택할 수 있게 해주는 PostgreSQL의 강력한 기능