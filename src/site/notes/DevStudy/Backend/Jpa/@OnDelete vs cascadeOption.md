---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Jpa/@OnDelete vs cascadeOption/","noteIcon":"","created":"2025-12-03T14:52:49.426+09:00","updated":"2025-12-13T10:31:29.233+09:00"}
---


## 1.  @OnDelete란?
하이버네이트의 고유 애노테이션 (JPA 표준)
데이터베이스 레벨의 CASCADE 동작을 지정하는데 사용된다.
즉, 이를 인지하는 것은 JPA레벨이 아니라 DB 그 자체이다.

작동 방식 
- DDL 외래 키 제약조건에 ON DELETE CASCADE or ON DELETE SET NULL 등의 속성을 추가하도록 하이버네이트에게 지시한다.
- 즉, cascade옵션과 달리 DDL 생성 시 스키마에 영향을 주는 것

**옵션 - `@OnDelete(action = )`**
- **기본 값** : @OnDelete해도 아무것도 하지 않는다. (따라서 action 지정 필요)
- **`@OnDelete(action = OnDeleteAction.CASCADE)`**
	- 부모 레코드 삭제 시 자식 레코드도 **데이터베이스가 직접 함께 삭제**. (SQL: `ON DELETE CASCADE`)
- **`@OnDelete(action = OnDeleteAction.RESTRICT)`**
	- 자식 레코드가 존재하는 경우 부모 레코드를 삭제할 수 없다.
- **`@OnDelete(action = OnDeleteAction.SET_NULL)`**
	- 부모 레코드 삭제 시 자식 레코드의 외래 키 값을 `NULL`로 설정합니다.

## 2.  단점 💢

**1차 캐시 불일치 문제 (가장 큰)**
- OnDelete는 데이터베이스가 직접 연쇄 삭제를 수행하도록 한다.
- 근데 JPA의 영속성 컨텍스트(1차 캐시)는 DB의 변경사항을 알지 못한다.
- 즉, 부모 엔티티를 삭제해서 DB의 자식 데이터는 삭제됐지만 트랜잭션이 끝나지 않는 한 **자식 엔티티는 1차 캐시에 남아있다.** 만약, 이후 로직에서 1차 캐시 데이터를 다루게 된다면 DB에서 **존재하지 않는 데이터를 다루게 되어서 논리적인 오류나 의도치 않는 동작을 유발**할 수 있다. 


>[!tip] OnDelele가 대용량 삭제시 강력한 성능최적화 기능이지만 애플리케이션 레벨에서의 많은 제어와 관리가 힘들다는 단점이 있기에 주의해서 사용해야 한다.


## 3.  @OnDelete와 cascade = CascadeType.REMOVE의 차이점

| 특징         | `@OnDelete(action = OnDeleteAction.CASCADE)`  | `cascade = CascadeType.REMOVE`                                  |
| :--------- | :-------------------------------------------- | :-------------------------------------------------------------- |
| 표준         | 하이버네이트 전용 확장 기능                               | JPA 표준 기능                                                       |
| **작동 주체**  | **데이터베이스**                                    | **JPA 영속성 컨텍스트 (애플리케이션 레벨)**                                    |
| **DB 스키마** | DDL 생성 시 외래 키 제약 조건에 `ON DELETE CASCADE` 등 추가 | DB 스키마에는 영향을 주지 않음                                              |
| **쿼리 발생**  | 부모 삭제 시 **하나의 쿼리**로 DB가 연쇄 처리 (매우 효율적)        | 부모 삭제 쿼리 외에, 자식 개수만큼 **여러 개의 `DELETE` 쿼리** 발생 가능 (N+1 문제 발생 가능) |
| **성능**     | 대량 삭제 시 일반적으로 **더 빠름** (DB 레벨 처리)             | 대량 삭제 시 N+1 쿼리 때문에 **상대적으로 느릴 수 있음**                            |
| **단점**     | 1차 캐시 불일치                                     | N+1                                                             |




