---
{"dg-publish":true,"permalink":"/DevStudy/AI/LangChain Agent 기본 설계 연습 겸 분석 (2) - 메모리, 라우터/","noteIcon":"","created":"2025-12-14T12:34:31.531+09:00","updated":"2025-12-14T13:27:53.493+09:00"}
---



> LangGraph를 활용한 AI 에이전트 만들기


## 1.  chat_message_histories 모듈 이용

### 1.1.  개념 및 구현체 
> 채팅 히스토리 백엔드들을 한 군데서 모아서 필요할 때만 로딩하는 **허브 모듈**

```python
from langchain_community.chat_message_histories import ChatMessageHistory
```

- *공통 인터페이스* : `BaseChatMessageHistory`
- *구현체* - `xxxChatMessageHitory`
	- 여기에는 `Redis`, `Postgres`, `File`, `InMemory` 등 다양한 구현체들이 있다
	  
| 구현체                        | 저장 위치            | 서버 재시작해도 유지? |
| -------------------------- | ---------------- | ------------ |
| ChatMessageHistory         | Python 메모리       | ❌            |
| RedisChatMessageHistory    | Redis            | ✔            |
| SQLChatMessageHistory      | PostgreSQL/MySQL | ✔            |
| PostgresChatMessageHistory | Postgres         | ✔            |
| MongoDBChatMessageHistory  | MongoDB          | ✔            |
| FileChatMessageHistory     | 로컬 파일            | ✔ (단일 서버)    |

- 즉, 이 모듈은 여러 구현체를 한 군데로 모으는 **Facade 패턴가 비슷** 
	- 이런 구조로 인해 저장소를 바꿔도 Agent로직은 거의 수정 없음 
- *Lazy Loader*
	- 실제로 이 구현체들이 import될 때 **동적으로 로딩**

>[!tip] 이런 모듈이 제공하는 구현체 덕분에 대화의 맥락을 유지하고 후속 질문이 가능해진다.


---
### 1.2.  인메모리 저장소 구현체 - ChatMessageHistory

> 서버 프로세스 메모리에 Python list 형태로 메시지를 저장하는 "기본 메모리 저장소"
```python
from langchain_community.chat_message_histories import ChatMessageHistory
```
매우 단순한 저장소 

아래는 사용자마다 독립된 채팅 히스토리 저장소 인스터를 만드는 예시 
```python
from langchain_community.chat_message_histories import ChatMessageHistory

store = {}


def get_session_history(session_id: str):
    """
    session_id에 해당하는 대화 기록 반환
    없으면 새로 만들기
    """
    if session_id not in store:
        store[session_id] = ChatMessageHistory()

    return store[session_id]
```
- 세션 단위로 대화 기록 분리
- 단일 서버/개발 환경에 적합 
- 실 서비스에서는 Redis / DB로 교체 


### 1.3.  다음 단계 

- 이렇게 쌓인 **대화 히스토리는 Router의 입력 데이터**가 된다.  
- 이를 활용해 **LLM 기반 의도 분류기(Router Chain)** 를 설계

---
## 2.  라우터 체인 - LLM 기반 의도 분류기 

### 2.1.  개념 
> 질문이 무슨 타입인지 분류해서 메인 Agent에게 넘겨주는 작은 AI 

- 메시지가 어떤 성격인지 어떤 의도인지 LLM으로 분류해주는 것이 router_chain이다
- Main Agent로 넘어가기 전에 Tag를 붙여줌 

```python
# Pydantic 모델을 LLM에 강제하는 라우터 체인 생성
router_chain = router_prompt | global_llm.with_structured_output(IntentClassifier)

raw = router_chain.invoke(
		{"input": request.query, "chat_history": full_history.messages}
)
```
`|` ❓
- LangChain Runnable 사이의 **파이프라인 연결을 만드는 연산자** 
	- 1의 출력 ➡ 2의 입력으로 연결 
	  
- 주의 : 비트 연산 아님 💢

---
### 2.2.  프롬프트 관련 패키지
> 챗 모델용 프롬프트 템플릿을 가지고 있는 패키지를 이용할 것 

```PYTHON
from langchain_core.prompts

router_prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    ("human", "{input}"),
    MessagesPlaceholder("chat_history"),
])
```
- system : 고정 지침
- human : 입력 
- history : 과거 대화 맥락 

>[!QUESTION] 프롬프트란❓ 
>간단히, LLM에게 던지는 입력이다.

- 프롬프트를 짤 때 단순한 문자열이 아니라 시스템 메시지, 유저 메시지, 예시 등 여러 컴포넌트를 조합하는 경우가 많다.
- 이런 **복잡한 프롬프트를 만들고 다루기 위한 클래스/함수들의 묶음**이 이 패키지이다.

종류
```PYTHON
__all__ = (
    "AIMessagePromptTemplate",
    "BaseChatPromptTemplate",
    "BasePromptTemplate",
    "ChatMessagePromptTemplate",
    "ChatPromptTemplate",
    "DictPromptTemplate",
    "FewShotChatMessagePromptTemplate",
    "FewShotPromptTemplate",
    "FewShotPromptWithTemplates",
    "HumanMessagePromptTemplate",
    "MessagesPlaceholder",
    "PromptTemplate",
    "StringPromptTemplate",
    "SystemMessagePromptTemplate",
    "aformat_document",
    "check_valid_template",
    "format_document",
    "get_template_variables",
    "jinja2_formatter",
    "load_prompt",
    "validate_jinja2",
)
```

---
### 2.3.  프롬프트 템플릿 활용 - `ChatPromptTemplate`
> 여러 개의 채팅 메시지(ai, human, system, placeholder 등)를 템플릿 형태로 묶고, 나중에 변수만 넣어서 message 리스트로 뽑아내는 빌더 

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

router_prompt = ChatPromptTemplate.from_messages(
    [
        ("system", "You are a helpful assistant."),
        ("human", "{input}"),
        MessagesPlaceholder("chat_history"),
    ]
)
```
- 여러 메시지를 조합해서 하나의 LLM 입력으로 만드는 템플릿 빌더
- `system` : 항상 고정된 시스템 메시지
- `human` : 유저가 새로 입력한 한 줄 
- `MessagsPlaceholder` : 과거 대화 히스토리 

> 이 구조를 하나의 템플릿 객체로 캡슐화 했음 


---
### 2.4.  IntentClassifier로 출력 강제
기존에 만든 router_prompt와 llm을 기반으로 router_chain을 만들 것 
```python
	class IntentClassifier(BaseModel):
    """
    라우터 AI가 반환할 Pydantic 모델 (JSON 양식) 정의
    """
    # Literal = 정해진 문자열 중 하나라는 타입 힌트 
    intent: Literal["NEW_SEARCH", "REFINEMENT", "CONVERSATION"] = Field(
        description=(
            "Classify as 'NEW_SEARCH' if user....).\n"
            "Classify as 'REFINEMENT' if ....').\n"
            "Classify as 'CONVERSATION' ...')."
        )
    )
```
- 자유 텍스트 응답 ❌
- **구조화된 JSON 출력 강제**
- Agent 안정성의 핵심 포인트
- LLM의 출력을 반드시 IntentClassifier 스키마 형식으로 강제
	- 여기서는 Router의 출력을 강제하기 위한 것 (특정 의도만 반환하도록)

```python
# router_prompt를 오른쪽의 입력으로 전달 
router_chain = router_prompt | global_llm.with_structured_output(IntentClassifier)

raw = router_chain.invoke(
		{"input": request.query, "chat_history": full_history.messages}
)
```
- `with_structured_output`은 내부적으로 `RunnableSequecne` 객체를 생성하는데, 첫 인자로 router_prompt를 전달한다.


>[!tip] 라우트 과정 요약
>1. ChatPromptTemplate형태로 프롬프트 포맷 
>2. 그걸 LLM에 전달
>3. LLM은 `IntentClassifier` 형태로 강제
>4. 의도 분류 후 Main 에이전트에 전달


>[!tip] 명확한 Intent 분류는 AI 에이전트의 핵심 


>[!QUESTION] 라우터가 없다면?
>- 도구 호출 판단이 흔들림
>- 대화 맥락이 너무 많이 섞임 
>- 에이전트 안정성이 떨어짐


```scss
[유저 입력]
     ↓
 ┌──────────┐
 │ Router   │  ← IntentClassifier(JSON)
 │ (LLM)    │  → ("NEW_SEARCH", "REFINEMENT", "CONVERSATION")
 └──────────┘
     ↓
 ┌─────────────────────┐
 │ Worker Agent + Tools │
 │ (weather, places, etc.) 
 └─────────────────────┘
```

### 2.5.  LangGraph 시작 
[[DevStudy/AI/LangGraph 개념\|LangGraph 개념]]