---
{"dg-publish":true,"permalink":"/프로젝트/나만무/실시간 편집/Redis Cache 초기 로딩 성능 테스트/","noteIcon":"","created":"2025-12-11T15:05:31.204+09:00","updated":"2025-12-11T15:10:56.007+09:00"}
---



## 1.  개요

> Redis Cache를 도입하여 초기 로딩 시간을 얼마나 개선했는지 측정한 성능 테스트

- **목표**: DB 직접 조회 vs Redis Cache 조회의 응답 시간 비교
- **측정 대상**: 워크스페이스 초기 진입 시 POI 데이터 로딩 시간
- **기대 효과**: Redis Cache 도입으로 인한 초기 로딩 시간 단축

## 2.  테스트 환경

### 2.1.  인프라
- **Database**: AWS RDS PostgreSQL 
- **Cache**: Elastic for Cache(Redis)

### 2.2.  측정 조건
- **반복 횟수**: 각 시나리오당 20회
- **측정 데이터**: 실제 프로덕션 워크스페이스 (POI 10개)
- **측정 단위**: 밀리초(ms)

## 3.  테스트 시나리오
### 3.1.  DB 직접 조회 (Cache Miss)
```sql

SELECT
  p.id,
  p.place_name,
  p.latitude,
  p.longitude,
  p.address,
  p.status,
  p.sequence,
  p.created_by,
  pd.id as plan_day_id,
  pl.id as place_id

FROM poi p
INNER JOIN plan_day pd ON p.plan_day_id = pd.id
LEFT JOIN places pl ON p.place_id = pl.id
WHERE pd.workspace_id = $1
ORDER BY pd.plan_date, p.sequence
```
- 매 반복마다 Redis 캐시를 비움
- **RDS에서 직접 쿼리 실행**
- **JOIN 연산** 포함 (plan_day, places 테이블)

### 3.2.  Redis 캐시 조회 (Cache Hit)

```typescript
// Redis Hash 구조로 캐싱
const cacheKey = `workspace:${workspaceId}:poi`;
const cachedPois = await redisClient.hVals(cacheKey);
return cachedPois.map((poi) => JSON.parse(poi));
```

- 사전에 데이터를 Redis에 캐싱 (워밍업)
  

## 4.  테스트 결과

### 4.1.  성능 비교표

| 측정 항목        | DB 직접 조회 | Redis 캐시 조회 | 개선율          |
| ------------ | -------- | ----------- | ------------ |
| **평균 응답 시간** | 4.57ms   | 0.20ms      | **95.5% 단축** |
| **표준편차**     | 0.76ms   | 0.11ms      | 85.5% 감소     |
| **최소 시간**    | 4.13ms   | 0.09ms      | 97.8% 단축     |
| **최대 시간**    | 7.78ms   | 0.50ms      | 93.6% 단축     |

평균 응답 시간이 22.37배 빠른 응답 속도
  

## 5.  핵심 성과
  
### 5.1.  응답 시간 95.5% 단축
- **개선 전**: 4.57ms (DB 직접 조회)
- **개선 후**: 0.20ms (Redis 캐시)
- **절대 시간 절약**: 4.37ms

### 5.2.  22.37배 빠른 응답 속도
- Redis 캐시를 사용하면 **약 22배 빠른 응답** 제공
- 사용자 경험 측면에서 **즉각적인 로딩** 구현

### 5.3.  안정적인 성능

- 표준편차가 0.76ms → 0.11ms로 감소
- **일관된 응답 시간** 보장


## 6.  결론

Redis Cache 도입으로 **초기 로딩 시간을 약 95.5% 단축**하여 사용자에게 **즉각적인 응답**을 제공할 수 있게 되었습니다. 특히 멀티 서버 환경에서도 Redis Pub/Sub을 통한 이벤트 동기화로 **일관된 실시간 협업 경험**을 보장합니다.