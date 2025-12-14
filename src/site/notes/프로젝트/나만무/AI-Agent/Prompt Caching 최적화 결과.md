---
{"dg-publish":true,"permalink":"/프로젝트/나만무/AI-Agent/Prompt Caching 최적화 결과/","noteIcon":"","created":"2025-12-10T00:40:43.810+09:00","updated":"2025-12-14T13:38:27.660+09:00"}
---



관련 
- [[프로젝트/나만무/나만무 개인 구현 기능 정리\|나만무 개인 구현 기능 정리]]
- 참고 깃허브 : [깃허브 링크 - prompt cache 브랜치로](https://github.com/NaManMu-10th-team7/matetrip-ai)
- AWS Prompt Caching 공부 : [[프로젝트/나만무/AI-Agent/AWS Bedrock Caching Prompt\|AWS Bedrock Caching Prompt]]

# Prompt Caching 최적화 결과 

---
## 1.  Preview - 최종 성과 요약 
![프롬프트 캐싱 4.png](/img/user/supporter/image/%ED%94%84%EB%A1%AC%ED%94%84%ED%8A%B8%20%EC%BA%90%EC%8B%B1%204.png)
AI Agent의 시스템 프롬프트(약 26k tokens)가 Router–Agent–Tool 호출마다 반복 전송되는 구조로 인해 턴당 평균 34k tokens가 발생했다. Bedrock Prompt Caching을 적용하여 시스템 프롬프트를 최초 1회만 모델에 로드하도록 개선함으로써, 아래와 같은 결과를 얻었다.

| 지표                | 개선 전            | 개선 후                         | 개선율        |
| ----------------- | --------------- | ---------------------------- | ---------- |
| 평균 Input Tokens/턴 | 약 34,000 tokens | 약 8,000 tokens (+ 26,000 캐시) | **76% 절감** |
| 턴당 비용             | $0.03829        | $0.0121                      | **68% 절감** |
| Cache Hit율        | 0%              | 100% (2턴부터)                  | 완전 고정      |

## 2.  기술 선택 배경 
LangGraph 기반 Agent는 한 턴 안에서도 Router → Agent → Tool → Post Processor 등 여러 LLM 호출을 수행한다.  

시스템 프롬프트가 매우 크기 때문에, 이 블록이 계속해서 재전송되는 구조는 비용·성능 측면에서 비효율적이었다.

---
## 3.  최적화 접근 방식


> AWS Bedrock Prompt Caching 이용 [[프로젝트/나만무/AI-Agent/AWS Bedrock Caching Prompt\|AWS Bedrock Caching Prompt]]

![프롬프트 캐싱2.png](/img/user/supporter/image/%ED%94%84%EB%A1%AC%ED%94%84%ED%8A%B8%20%EC%BA%90%EC%8B%B12.png)
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

### 4.1.  단가 확인 
[출처](https://aws.amazon.com/ko/bedrock/pricing/)

| **Anthropic models** | **Price per 1,000 input tokens** | **Price per 1,000 output tokens** | **Price per 1,000 input tokens (batch)** | **Price per 1,000 output tokens (batch)** | **Price per 1,000 input tokens (cache write)** | **Price per 1,000 input tokens (cache read)** |
| -------------------- | -------------------------------- | --------------------------------- | ---------------------------------------- | ----------------------------------------- | ---------------------------------------------- | --------------------------------------------- |
| Claude Haiku 4.5     | $0.0011                          | $0.0055                           | $0.00055                                 | $0.00275                                  | $0.001375                                      | $0.00011                                      |


### 4.2.  Token 사용량 변화

*💢개선 전* 
- **매 요청마다 26k tokens 시스템 프롬프트 재전송**
- 평균 32k~34k tokens/턴

*✅개선 후(Prompt Caching 적용)*
- 턴 1 : Input: 425 tokens + Cache write: 26K tokens
- 턴 2 : Input: 8.4K tokens + Cache read: 26K tokens
- 턴 3 : Input: 8.6K tokens + Cache read: 26K tokens

> Input 기준 약 76% 절감 

### 4.3.  비용 계산 전 단가 확인 
>[!tip] Bedrock Claude Haiku 단가
>- input: $0.0011 per 1K
>- Cache write: $0.001375 per 1K
>- Cache read: $0.00011 per 1K ✅
>- Cache에 쓰는 비용보다 읽는 비용이 90%는 싸다 

### 4.4.  3턴 기준 비용 계산 

#### 4.4.1.  개선 전

- 턴 1: 26.798K × $0.0011 = **$0.02948**
- 턴 2: 34.812K × $0.0011 = **$0.03829**
- 턴 3: 34.975K × $0.0011 = **$0.03847**
- **Total = $0.10624**

#### 4.4.2.  개선 후
- 턴 1: (0.425K × $0.0011) + (26K × $0.001375) = $0.00047 + $0.03575 = **$0.03622**
- 턴 2: (8.4K × $0.0011) + (26K × $0.00011) = $0.00924 + $0.00286 = **$0.01210**
- 턴 3: (8.6K × $0.0011) + (26K × $0.00011) = $0.00946 + $0.00286 = **$0.01232**
- **Total = $0.06064**

> **총 42% 절감 (3턴 기준)**

> 대화가 길어질수록 cache write의 초기 비용이 분산되어 절감율 증가

---
## 5.  요약 

- 시스템 프롬프트 블록(26k tokens)을 LLM KV Cache에 고정
- Router ~ Agent 전체 호출이 동일 캐시를 사용
- Input token 76% 감소 / 비용 68% 절감
- 장기 대화에서 Cache Hit 100% 유지