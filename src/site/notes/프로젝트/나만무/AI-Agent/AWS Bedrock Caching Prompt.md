---
{"dg-publish":true,"permalink":"/프로젝트/나만무/AI-Agent/AWS Bedrock Caching Prompt/","noteIcon":"","created":"2025-12-09T17:19:41.489+09:00","updated":"2025-12-14T14:33:41.209+09:00"}
---




- Bedrock Prompt Caching은 LLM 요청 단계에서 시스템 프롬프트를 ‘토큰으로 다시 펼쳐서 파싱하지 않는 방식으로 설계되어 있어 캐싱된 프롬프트는 **토큰 비용을 거의 0에 가깝게** 처리하도록 설계되어 있음.

- 참고 자료 
	- [AWS 공식 홈페이지](Amazon Bedrock Prompt Caching with the Amazon Nova Model)
	- [AWS Blog](https://aws.amazon.com/ko/blogs/machine-learning/effectively-use-prompt-caching-on-amazon-bedrock/)
	- [사용 설명서](https://docs.aws.amazon.com/ko_kr/bedrock/latest/userguide/prompt-caching.html?utm_source=chatgpt.com)
	- [Medium 프롬프트 캐싱](https://medium.com/@nikitaparate9/prompt-caching-in-llm-systems-db1a53dd671f)


## 1.  Prompt Caching이란?

### 1.1.  개념 
> 같은 프롬프트로 자주 호출되는 LLM 요청에서, **프롬프트 부분만 캐시에 저장해 두고 재사용해서 비용/지연을 크게 줄이는 기능**


### 1.2.  필요한 이유 

일반적인 LLM 호출은 **매 요청마다 전체 프롬프트를 처음부터 다시 처리**한다.

즉, 매 요청마다 다음 비용이 반복된다.
- 토크나이징(Tokenization)
- Embedding
- Attention용 **KV Cache 생성**
- Transformer Layer 전체 통과

하지만 실제 서비스에서는:
- **System Prompt**
- **정책 / 규칙 / 포맷 지시**
- **도메인 설명**

같은 **변하지 않는 긴 프롬프트**가 매 요청마다 반복된다.

이 반복 비용을 제거하기 위해 등장한 구조가 **Prompt Caching**

### 1.3.  원리 
변하지 않는 길고 무거운 System Prompt·도구 설명·예시 등은 **한 번만 계산해 저장**
- 이후 동일한 프롬프트 조각이 들어오면 **캐시된 결과를 불러와 재계산을 건너뜀**
- 그 결과 **입력 토큰 비용 감소**, **추론 지연 단축** 효과가 발생
- 컨텍스트의 일부를 추가하면 모델은 캐시를 활용하여 입력의 재계산을 건너뛰어 Bedrock이 컴퓨팅 절감액을 공유하고 응답 지연 시간을 줄일 수 있다.
- 추론 응답 지연 시간과 입력 토큰 비용을 줄일 수 있다.


### 1.4.  언제 도움 되는가❓


여러 쿼리에 자주 재사용되는 길고 반복적은 context가 있는 경우 도움 


### 1.5.  가격 확인
[출처](https://aws.amazon.com/ko/bedrock/pricing/)
일단 AWS Bedrock에서 써본게 Claude 3.5, 4.5 랑 Nova계열인데 그거 위주로 ㄱ 

| **Anthropic models** | **Price per 1,000 input tokens** | **Price per 1,000 output tokens** | **Price per 1,000 input tokens (batch)** | **Price per 1,000 output tokens (batch)** | **Price per 1,000 input tokens (cache write)** | **Price per 1,000 input tokens (cache read)** |
| -------------------- | -------------------------------- | --------------------------------- | ---------------------------------------- | ----------------------------------------- | ---------------------------------------------- | --------------------------------------------- |
| Claude Haiku 4.5     | $0.0011                          | $0.0055                           | $0.00055                                 | $0.00275                                  | $0.001375                                      | $0.00011                                      |
| Claude 3.7 Sonnet    | $0.003                           | $0.015                            | $0.0015                                  | $0.0075                                   | $0.00375                                       | $0.0003                                       |
| Claude 3.5 Sonnet v2 | $0.003                           | $0.015                            | $0.0015                                  | $0.0075                                   | $0.00375                                       | $0.0003                                       |
|                      |                                  |                                   |                                          |                                           |                                                |                                               |
참고 : Region마다 가격이 다름 


## 2.  과정 - 그림 설명 

> LLM은 앞으로 특정 부분을 매번 다시 계산하지 않아도 된다.

계산된 상태를 그대로 가져오는 방식이야.
즉, RE-computation이 아니라 **state reuse(상태 재사용)**.


### 2.1.  전체 그림 
![Pasted image 20251211163435.png](/img/user/supporter/image/Pasted%20image%2020251211163435.png)


### 2.2.  Case 1 - prefix 부분이 완전히 같은 경우 
Prefix 부분 `A B C`가 완전히 같음 → **Cache Hit**
- 앞 부분은 계산을 Pass
- LLM은 E, F만 새로 계산하면 됨

### 2.3.  Case 2 - 중간 prefix 달라지는 



### 2.4.  Case - Prefix는 똑같지만 다른 순서가 변경된 경우
> A B C가 포함되어있지만 **앞부분이 동일해야 캐시가 작동한다**.


즉, **Prefix가 이동하거나 순서가 바뀌면 → Cache Miss**




## 3.  LLM의 토큰 비용 계산 과정

일반 LLM 요청은 다음 과정을 거쳐:
1. **프롬프트 전체를 토크나이저로 변환 → tokens list 생성**
2. 이 tokens 을 모델의 입력 embedding layer로 집어 넣음  
    → 메모리 배치(배치 사이즈 관리, KV Cache 생성 등)
    
3. attention 연산 수행
4. output 생성

즉, **토큰이 많을수록**
- 토크나이징 비용
- 패딩 비용
- KV 캐시 구축 비용
- attention compute 비용이 모두 증가.
    
➡ 그래서 시스템 프롬프트가 26k라면 26k 전체가 매턴마다 처리됨.

> [!INFO] KV(Key-Value) Cache
> Transformer 기반 LLM이 **이미 처리한 토큰들의 attention 계산 결과를 저장해 두는 메모리 구조**


---

## 4.  Prompt Caching은 이 전체 과정을 건너뛰는 구조

AWS가 발표한 Prompt Caching 설명에는 아래처럼 되어 있음

> 캐싱된 프롬프트는 다시 처리할 필요가 없다

이 말은 **"프롬프트 토큰 전체를 모델이 다시 임베딩·파싱하지 않는다"** 는 뜻.

다시 말해,
- 토크나이징 ❌
- KV(Key-Value) 캐시 생성 ❌
- attention layer에서 baseline token으로 계산 ❌

그 대신 “이미 계산된 KV Cache 조각”을 바로 불러서 이어붙여 모델 연산 시작하는 방식


