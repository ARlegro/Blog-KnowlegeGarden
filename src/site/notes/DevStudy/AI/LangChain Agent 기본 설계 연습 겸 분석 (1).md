---
{"dg-publish":true,"permalink":"/DevStudy/AI/LangChain Agent 기본 설계 연습 겸 분석 (1)/","noteIcon":"","created":"2025-12-03T14:53:07.076+09:00","updated":"2025-12-14T12:42:39.777+09:00"}
---



LLM을 단순히 호출하는 것과 **AI Agent를 설계하는 것**은 전혀 다른 문제
Agent는 단순 응답기가 아니라, **대화 맥락을 기억하고 상황에 따라 판단하며 외부 도구를 활용하는 시스템** (참고 : [[DevStudy/AI/AI Agent 기본 원리\|AI Agent 기본 원리]])


> LangChain 생태계를 기반으로 **LangGraph를 활용한 AI Agent 설계 방법**을 단계적으로 정리


---
## 1.  필요 패키지

### 1.1.  핵심 패키지 
```text
langchain  
langchain-community
```
- `langchain`
	- 랭체인 메인 라이브러리.
	- Chain, Prompt template, Runnable 등의 Agent 구성의 기본 추상화 제공 
- `langchain-community`
	- 커뮤니티에서 관리하는 다양한 서드파티 툴과 통합 기능을 제공. 
	- 검색 도구, DB, 캐시, 메모리 저장소 등 외부 세계와 연결되는 대부분의 기능이 여기에 포함
	- 에이전트가 외부 도구를 사용하려면 필수적

단순 LLM 호출만 한다면 `langchain`만으로도 가능하지만, **Agent는 거의 항상 `langchain-community`를 필요로 함.**

---
### 1.2.  LLM 연동 패키지
> 사용하려는 LLM에 따라 별도의 파트너 패키지 必

```python
# GPT사용 시 
langchain-openai

# Claude 사용 시 
langchain-anthropic

# aws에 있는 LLM 사용 시 
langchain-aws
```



---
### 1.3.  에이전트 오케스트레이션 ⭐(사실상 필수 표준)
> LangGraph : LangChain 팀에서 만든 최신 에이전트 프레임워크 

- 상태를 중심으로 Agent를 그래프 구조로 명시적으로 설계
- 순환 구조, 분기, 재시도, 조건부 흐름에 강함 
- 복잫반 Agent 구축 시 표준

💢레거시  방식 예시 
```python
from langchain.agents import create_react_agent
from langchain_classic.agents import AgentExecutor, create_tool_calling_agent
```
- 상태 관리 어렵고
- 복잡한 흐름 표현이 어려움 
- 따라서, 단순 데모용이고 실무 설계에서는 한계 명확 

---
### 1.4.  도구 및 검색 
> 에이전트가 외부 세상의 정보를 검색하거나 계산하기 위해 필요한 패키지들

- **`tavily-python`**: AI 검색 엔진인 Tavily API 연동 (웹 검색 도구로 가장 많이 쓰임).
- **`duckduckgo-search`**: 무료 웹 검색 도구.
- **`numexpr`**: 수학 계산 에이전트를 만들 때 필요.

---
### 1.5.  백터 DB 관련(메모리 & RAG)
> 에이전트에게 장기 기억을 주거나 문서를 참조(RAG)하게 하려면 필요한 패키지 

- **`langchain-chroma`** 또는 **`faiss-cpu`**: 벡터 저장소 라이브러리
- **`langchain-text-splitters`**: 텍스트를 쪼개는 유틸리티

---
### 1.6.  보조 패키지 
```TEXT 
fastapi
uvicorn[standard]
boto3
httpx
python-dotenv
```

---
## 2.  관련 개념 미리보기 

### 2.1.  Router 
> LLM이 다음에 어떤 단계를 취할 지 결정하는 역할

- 사용자의 질문이
    - 새로운 검색인지
    - 기존 결과의 수정인지
    - 단순 대화인지  
        를 먼저 분류
        

이 Router가 **Agent 안정성의 핵심**

다음 : [[DevStudy/AI/LangChain Agent 기본 설계 연습 겸 분석 (2) - 메모리, 라우터\|LangChain Agent 기본 설계 연습 겸 분석 (2) - 메모리, 라우터]]

---