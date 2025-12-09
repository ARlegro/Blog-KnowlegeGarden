---
{"dg-publish":true,"permalink":"/프로젝트/나만무/Prompt Caching 최적화 결과/","noteIcon":"","created":"2025-12-10T00:40:43.810+09:00","updated":"2025-12-10T01:11:04.299+09:00"}
---



관련 
- [[나만무 담당 부분 정리\|나만무 담당 부분 정리]]


> 참고 내용 : [Prompt-Caching test 브랜치](https://github.com/NaManMu-10th-team7/matetrip-ai)

# Prompt Caching 최적화 결과 

---
## 1.  Preview - 최종 성과 요약 

AI Agent의 시스템 프롬프트(약 26k tokens)가 Router–Agent–Tool 호출마다 반복 전송되는 구조로 인해 턴당 평균 34k tokens가 발생했다. Bedrock Prompt Caching을 적용하여 시스템 프롬프트를 최초 1회만 모델에 로드하도록 개선함으로써, 아래와 같은 결과를 얻었다.

| 지표                | 개선 전            | 개선 후                         | 개선율        |
| ----------------- | --------------- | ---------------------------- | ---------- |
| 평균 Input Tokens/턴 | 약 34,000 tokens | 약 8,000 tokens (+ 26,000 캐시) | **76% 절감** |
| 턴당 비용             | $0.0087         | $0.0028                      | **68% 절감** |
| Cache Hit율        | 0%              | 100% (2턴부터)                  | 완전 고정      |

## 2.  기술 선택 배경 
LangGraph 기반 Agent는 한 턴 안에서도 Router → Agent → Tool → Post Processor 등 여러 LLM 호출을 수행한다.  

시스템 프롬프트가 매우 크기 때문에, 이 블록이 계속해서 재전송되는 구조는 비용·성능 측면에서 비효율적이었다.

---
## 3.  최적화 접근 방식


> AWS Bedrock Prompt Caching 이용 [[Bedrock Caching Prompt\|Bedrock Caching Prompt]]

1. *System Prompt Caching 도입*
	- **시스템 프롬프트(26k tokens)를 캐시에 저장**
	- 첫 요청에서만 `cache write`
	- 이후 모든 요청에서 `cache read`로 대체 → 모델 계산 스킵
2. *Router/Agent 전체 구간에 캐시 공유*
	- 한 세션 내 모든 노드에서 동일 캐시 세그먼트를 재사용


3. **구현 위치**
	``` python
	LangGraph Agent -> (app/agent/builder2.py)
	LangGraph Router -> (app/agent/nodes/router_node.py) 
	```
 4. **코드 예시** 
	 ```python
     # AWS Bedrock Prompt Caching 활성화	 
     cached_system_message = SystemMessage(
        content=[
            {
                "type": "text",
                "text": main_system_prompt,
            },
            {
                "cachePoint": {
                    "type": "default", 
                    ...
	 ```

---
## 4.  성과 상세 분석 
### 4.1.  Token 사용량 변화

*💢개선 전* 
- **매 요청마다 26k tokens 시스템 프롬프트 재전송**
- 평균 32k~34k tokens/턴

*✅개선 후(Prompt Caching 적용)*
- 턴 1 : Input: 425 tokens + Cache write: 26K tokens
- 턴 2 : Input: 8.4K tokens + Cache read: 26K tokens
- 턴 3 : Input: 8.6K tokens + Cache read: 26K tokens

> Input 기준 약 76% 절감 

### 4.2.  비용 계산 전 단가 확인 
>[!tip] Bedrock Claude Haiku 단가
>- input: $0.00025 per 1K
>- Cache write: $0.0003125 per 1K
>- Cache read: $0.000025 per 1K ✅
>- Cache에 쓰는 비용보다 읽는 비용이 90%는 싸다 

### 4.3.  3턴 기준 비용 계산 

#### 4.3.1.  개선 전

- 턴 1: 26.798k × $0.00025 = $0.0067 
- 턴 2: 34.812k × $0.00025 = $0.0087 
- 턴 3: 34.975k × $0.00025 = $0.0087 
- **Total = $0.0241**

#### 4.3.2.  개선 후
- 턴 1: Input + Cache write = $0.0083 
- 턴 2: $0.0028 
- 턴 3: $0.0029 
- **Total = $0.014** 

> **총 42% 절감 (3턴 기준)**

> 대화가 길어질수록 cache write의 초기 비용이 분산되어 절감율 증가

---
## 5.  요약 

- 시스템 프롬프트 블록(26k tokens)을 LLM KV Cache에 고정
- Router ~ Agent 전체 호출이 동일 캐시를 사용
- Input token 76% 감소 / 비용 68% 절감
- 장기 대화에서 Cache Hit 100% 유지