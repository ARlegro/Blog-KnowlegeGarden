---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Redis/Setting/더미 데이터 만들기(MySQL, PostgreSQL)/","noteIcon":"","created":"2025-12-03T14:52:50.243+09:00","updated":"2025-12-13T10:24:06.337+09:00"}
---



### 0.1.  mysql 
```SQL
SET SESSION cte_max_recursion_depth = 1000000;  
  
-- boards 테이블에 더미 데이터 삽입  
INSERT INTO boards (title, content, created_at)  
WITH RECURSIVE cte (n) AS  
                   (  
                       SELECT 1  
                       UNION ALL  
                       SELECT n + 1 FROM cte WHERE n < 1000000 -- 생성하고 싶은 더미 데이터의 개수  
                   )  
SELECT  
    CONCAT('Title', LPAD(n, 7, '0')) AS title,  -- 'Title' 다음에 7자리 숫자로 구성된 제목 생성  
    CONCAT('Content', LPAD(n, 7, '0')) AS content,  -- 'Content' 다음에 7자리 숫자로 구성된 내용 생성  
    TIMESTAMP(DATE_SUB(NOW(), INTERVAL FLOOR(RAND() * 3650 + 1) DAY) + INTERVAL FLOOR(RAND() * 86400) SECOND) AS created_at -- 최근 10년 내의 임의의 날짜와 시간 생성  
FROM cte;
```
단순 이 Script로도  더미데이터 엄청 생김 


## 1.  postgreSQL 버전 
https://neon.com/postgresql/postgresql-tutorial/postgresql-generate_series?utm_source=chatgpt.com


간단한 generate_series() 사용 

```SQL
INSERT INTO boards (title, content, created_at)
SELECT
    'Title' || LPAD(n::text, 7, '0'),
    'Content' || LPAD(n::text, 7, '0'),
    NOW() - (INTERVAL '1 day' * FLOOR(random() * 3650 + 1))
         + (INTERVAL '1 second' * FLOOR(random() * 86400))
FROM generate_series(1, 1000000) AS s(n);
```


### 1.1.  코드 설명 

1. `FROM generate_series(1, 1000000) AS s(n)`
	- 가상 테이블 생성 
	- 1부터 1,000,000까지의 정수를 가진 1,000,000행짜리 가상 테이블 생성
	- **별칭(`AS s(n)`)**:
		- `s` : 가상 테이블의 이름 
		- (n) : 열의 이름 
	  
2. **n::text** 
	- `n`은 `generate_series`로부터 생성된 정수형 값
	- `::text`는 PostgreSQL의 **형 변환(casting)** 문법
	- ex. 123:text ➡ '123'
	  
3. **LPAD(문자열, 길이, 채울문자)**
	- 주어진 문자열을 왼쪽(`Left`) 쪽에서부터 특정 길이만큼 `채울문자`로 채워준다.
	- ex. LPAD(42, 7, '0') ➡ '0000042'  
	- 반드시 길이만큼 맞추는 효과 
4. `| |`  : 문자열 연결 
	  
5. **`NOW() - (INTERVAL '1 day' * FLOOR(random() * 3650 + 1)) + (INTERVAL '1 second' * FLOOR(random() * 86400))`**
	- `random()` : PostgreSQL 내장 함수. **0.0 이상 1.0 미만**의 부동소수점을 균등 분포로 반환
	- `random() * 3650 + 1` ➡ 1부터 3650까지 임의의 값 생성 생성
	- Floor : 소수점 버리고 정수로 만듬 
	- INTERVAL '1 day' : '1일'이라는 기간(Interval)
	- INTERVAL '1 second' : '1초'이라는 기간(Interval)
		

>[!tip] 만약 랜덤한 숫자 생성하고 싶다면 ➡ ORDER BY random()  +  서브 쿼리
```sql 
FROM (
    SELECT n
    FROM generate_series(1, 1000000) AS s(n)
    ORDER BY random()
) AS shuffled;
```
- 1~1000000 사이의 반환된 숫자를 random으로 섞음 








>단일쿼리로 대량 데이터 즉시 생성 삽입 가능 

