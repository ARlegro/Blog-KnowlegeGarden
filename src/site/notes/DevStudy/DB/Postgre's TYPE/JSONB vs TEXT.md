---
{"dg-publish":true,"permalink":"/DevStudy/DB/Postgre's TYPE/JSONB vs TEXT/","noteIcon":"","created":"2025-12-09T17:19:47.457+09:00","updated":"2025-12-13T18:25:30.315+09:00"}
---




```java
@Column(name = "keywords", columnDefinition = "jsonb")  
@JdbcTypeCode(SqlTypes.JSON)  
private List<String> keywords;
```

위는 이전 팀원이 만든 코드이고, SQL에서는 JSONB타입을 썼다

하지만 PostgreSQL은 `TEXT[]` 가 있어서 둘의 차이를 알아볼까 한다.


### 0.1.  네이티브 배열 타입 (`text[]`)


```SQL 
CREATE TABLE interests 
(
		id        UUID      PRIMARY KEY,   
		name      VARCHAR(50) NOT NULL UNIQUE,   
		keywords  TEXT[]      NULL,   
		created_at TIMESTAMPTZ NOT NULL DEFAULT now() 
);
```

- **장점**    
    - 스키마가 간단해지고, ORM 매핑(예: Hibernate Types)으로 `List<String>` ↔ `text[]` 매핑이 쉽습니다.
    - GIN 인덱스(`CREATE INDEX ON interests USING GIN(keywords)`)를 걸어 `@>`, `&&` 같은 배열 연산자 성능을 최적화할 수 있습니다.
        
- **단점**
    - 키워드를 개별 테이블로 분리하지 않으므로, 각 키워드가 다른 엔티티에도 재사용될 때 중복 저장이 발생 (흠.... 완전 정규화까지???)
    - 배열 내 원소 타입이 모두 동일해야 하며, 복잡한 구조(객체 형태)는 담기 어렵다. ????


---

### 0.2.  JSONB 컬럼 (`jsonb`)

```SQL
CREATE TABLE interests 
(
		…,
	 keywords JSONB NULL 
);
```
- **장점**
    - 스키마 유연성이 높아, **단순 리스트 외에 객체·배열의 중첩 구조도 함께 저장**할 수 있습니다. (흠... 근데 객체를 쓸 일이 많이 없을 듯???)
    - `keywords @> '["park","finance"]'::jsonb` 같은 JSONB 연산자를 활용해 검색 가능하며, GIN 인덱스(`USING GIN`)도 지원됩니다.
        
- **단점**    
    - 내부 자료형이 “무타입(typeless)” JSON이므로, **단순 문자열 리스트를 다룰 때는 오버헤드**가 있다.
    - **`text[]` 대비 저장·파싱 성능이 조금 떨어질 수** 있다.


## 1.  변경 해보기

SQL을 JSONB ➡ TEXT[ ] 로 바꿨기에 자바도 바꿔볼 것 
기존에는 아래와 같았다.
```JAVA 
@Column(name = "keywords", columnDefinition = "jsonb")  
@JdbcTypeCode(SqlTypes.JSON)  
private List<String> keywords;
```

원래는 Hibernate 의 라이브러리를 추가해야 했었다.
하지만, Hibernate 6부터는 외부 라이브러리 없이도 `String[]` ↔ `text[]`를 바로 매핑할 수 있다
```JAVA
@Column(name = "keywords", columnDefinition = "text[]")  
@JdbcTypeCode(SqlTypes.ARRAY)  
private List<String> keywords;
```


