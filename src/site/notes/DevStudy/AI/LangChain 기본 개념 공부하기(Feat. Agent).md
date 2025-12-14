---
{"dg-publish":true,"permalink":"/DevStudy/AI/LangChain 기본 개념 공부하기(Feat. Agent)/","noteIcon":"","created":"2025-12-03T14:53:06.823+09:00","updated":"2025-12-10T13:53:44.329+09:00"}
---



> 나만무를 진행하면서 AI-Agent를 만들기 위한 거의 첫 공부 (완전 초창기라 미숙)

---
## 1.  사전 - Agent

---
### 1.1.  개념
>[!info] agent란❓
>- 언어 모델을 활용해 일련의 행동을 스스로 선택하게 만드는 것
>- Agent는 추론 엔진을 통해 어떤 행동을 할지, 어떤 순서로 진행할지를 스스로 결정
>- 기본 3요소
>	1. 계획
>	2. 기억
>	3. 도구 화룡

---
### 1.2.  왜 필요한가

LLM을 단순 “질문 → 답변” 생성기로 쓰면 할 수 있는 일이 제한된다. **실제 문제 해결 시스템**으로 만들려면 LLM이 아래 역할을 할 수 있어야 한다.
- 정보를 **기억**하고
- 필요한 작업을 여러 단계로 나누고 **스스로 결정**하고
- DB/검색/내부 API 같은 **외부 도구를 호출**하고
- **이전 결과를 기반**으로 다음 행동을 선택해야 함 

**이것들을 가능하게 하는 구조 ➡ Agent ⭐⭐**

---

### 1.3.  정리 

Agent는 
- 스스로 계획하고
- 기억을 활용하고
- 도구를 선택해서 문제 해결하는  

> 지능형 실행 시스템


---
## 2.  LangChain 개념

> LLM을 활용한 애플리케이션을 쉽게 만들 수 있도록 도와주는 프레임워크 

- ❌단순 프롬프트 ➡**응답** 용 라이브러리가 아니다
- ✅**도구 호출, 메모리 관리, 체인 구성, 에이전트 설계 등 복잡한 워크플로우 제작에 최적화** 

Agent라는 개념/아키텍쳐를 **코드로 구현하기 위한 프레임워크** 

---
## 3.  LangChain 핵심 개념들 

---
### 3.1.  Prompt Templates - 프롬프트 재사용
> LLM에 보낼 입력을 템플릿으로 관리하는 기능

```PYTHON
from langchain.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "너는 여행 추천 전문 비서야."),
    ("human", "사용자 목적지: {region}\n사용자 취향: {style}")
])
```
- `{region}`, `{style}` 같은 부분은 **런타임에 채워 넣는 변수**
- 프롬프트를 코드 곳곳에서 문자열로 관리하지 않고, **템플릿으로 정의 → 재사용**할 수 있게 도와준다.

---
### 3.2.  Chains - LLM 호출 + 로직을 파이프라인처럼 조립
LLM 호출 + 로직을 파이프라인처럼 조립하는 기능 
```python
# Chain 조립 (프롬프트 → LLM → 파서)
router_chain = router_prompt | global_llm.with_structured_output(IntentClassifier)
```

의도 구조화 예시 
```python
class IntentClassifier(BaseModel):
    intent: Literal["NEW_SEARCH", "REFINEMENT", "CONVERSATION", "FOLLOW_UP"] = Field(
        description=(
            "Classify as 'NEW_SEARCH' if user requests a completely new place search (e.g., 'Busan restaurants' after asking 'Seoul cafes').\n"
            ...
        )
```

`with_structured_output(IntentClassifier)` : 뒤에서 자세히 나옴 
- 아래와 같은 효과 있음
	- “너는 반드시 아래 Pydantic 모델의 JSON만 만들어라.  
	- 그 외 문장, 말투, 설명, 텍스트는 금지.”

➡ LLM은 자연어를 전혀 출력하지 못하고 **딱 이 JSON만 내놓게 된다.**

---
### 3.3.  Memory
> 이전 대화 내용/컨텍스트를 어떤 규칙에 따라 LLM에 넘길지를 관리하는 기능

메시지 History를 활용하는 여러 전략으로 사용 가능
- 최근 N개만 보여주던
- 요약본만 보여주던
- 특정 타입(AI, Human, Tool 등)의 메시지만 골라주던지

LangChain에서의 메모리 관리 기능
1. 구현체 활용 : `ConversationBufferMemory`, `ConversationSummaryMemory` 등
2. 직접 Memory : chat_history 필드에 끼워넣기 

```python
llm = ChatOpenAI(model="gpt-4o-mini")

# 1. 메모리 객체
memory = ConversationBufferMemory(
    return_messages=True  # chat_history를 메시지 리스트 형태로 유지
)

# 2. 대화용 Chain
conversation = ConversationChain(
    llm=llm,
    memory=memory,
    verbose=True,  # 내부에서 프롬프트 어떻게 보내는지 로그 출력
)
```
Memory를 Chain/Agent에 “주입" 

---
### 3.4.  Tools Agent 
> Tool = “LLM이 사용할 수 있는 외부 기능”을 추상화한 것

> Agent = “LLM + Tools + 프롬프트 + 메모리”를 합쳐서 **어떤 도구를 어떤 순서로 쓸지 스스로 결정하는 엔진**

```python
agent = create_tool_calling_agent(llm, tools, prompt)
```

---
### 3.5.  IntentClassifer ⭐
> 자연어로 “대화인지, 도구 호출인지”를 파싱하는 대신, **LLM에게 아예 JSON 스키마를 강제해서 받는 방식**


- AI가 만든 문장을 이용해 직접 분기하는 게 위험해서,  안전하게 정해진 JSON 형태로만 의도를 반환하도록 만든 모델.
- 단순 자연어보다는 엄격한 JSON으로 해야 기계가 이해하기 쉬움 
- **의도 라우팅을 안정적으로 하기 위한 장치**.
	- 분기처리를 통해 구조 라우팅 가능
		```PYTHON
		if intent == "CONVERSATION":
            # [기억 사용 O] 후속 질문으로 모든 기억을 다 줌
            history_to_pass = full_history.messages
        else:
            # [기억 사용 X] 사용자의 말만 줌
            for msg in full_history.messages:
		```

```PYTHON
class IntentClassifier(BaseModel):
    """
    라우터 AI가 반환할 Pydantic 모델 (JSON 양식) 정의
    """
    intent: Literal["TOOL_USE", "CONVERSATION"] = Field(
        description=(
            "Classify as 'TOOL_USE' if the user needs new info (search, create).\n"
            "Classify as 'CONVERSATION' for simple chat or follow-ups about the last response."
        )
    )
```
- AI가 `“TOOL_USE"`, `“CONVERSATION”` 외 다른 응답을 절대 생성하지 않음.  
	```PYTHON
	{
		  "intent": "CONVERSATION"
	}
	{
		  "intent": "TOOL_USE"
	}	
	```
	- 필드는 단 하나.
	- 이 필드의 목적 : 의도를 라우팅하는 기준 


>[!tip] 의도 분류의 핵심
>1. 정확한 JSON 생성
>2. 타입/옵션 강제
>3. 엉뚱한 답 생성 방지 핵심


---
## 4.  기본 AI Agent 라우팅 코드 분석 

---
### 4.1.  상세 코드와 단계 
아래는 AI-Agent 챗봇의 코드 일부이다.
```python
# Pydantic 모델을 LLM에 강제하는 라우터 체인 생성
router_chain = router_prompt | global_llm.with_structured_output(IntentClassifier)

class ChatRequest(BaseModel):
    query: str
    session_id: str

@router.post("/", response_model=ChatResponse)
async def ask_agent(request: ChatRequest) -> ChatResponse:
    """
    AI 에이전트 및 챗봇 실행 엔드포인트
    (대화 응답 + 구조화된 도구 데이터를 함께 반환)
    """
    try:
        # [라우터 실행] AI에게 사용자의 의도부터 물어봄
        full_history = get_session_history(request.session_id)

			  # LLM 라우터로 의도 분류 
        raw = router_chain.invoke(
            {"input": request.query, "chat_history": full_history.messages}
        )
        classification_result: IntentClassifier = IntentClassifier.model_validate(raw)
        intent = classification_result.intent

        logger.info(f"AI Router Intent: {intent}")

        # [코드 기반 분기] 의도에 따라 일꾼에게 전달할 기억 선별
        history_to_pass = []

        if intent == "CONVERSATION":
            # [AI 답변 사용 O] 후속 질문으로 모든 기억을 다 줌
            history_to_pass = full_history.messages
        else:
            # [AI 답변 기억 사용 X] 사용자의 말만 줌
            for msg in full_history.messages:
                if isinstance(msg, HumanMessage):
                    history_to_pass.append(msg)

        logger.info(f"history_to_pass : {history_to_pass}")

        # 1. 도구 생성
        user_tools = create_nest_tools()

        # 2. 에이전트 조립 (global_llm 재사용)
        agent: AgentExecutor = build_stateful_agent(global_llm, user_tools)

        # 3. 실행
        # 핵심 로직은 agent_service로 위임
        chatResponse: ChatResponse = get_agent_response(agent, request, history_to_pass)

        # 4. 대화 기록 수동 저장
        full_history.add_user_message(request.query)
        # full_history.add_ai_message(response_dict"response"])
        full_history.add_ai_message(chatResponse.response)
        return chatResponse

    except Exception as e:
        return ChatResponse(
            response=f"처리 중 오류가 발생했습니다. : {str(e)}", tool_data=[]
        )
```

1. *LLM 라우터로 의도 분류 - `router_chain.invoke`*
	- 사용자의 요청(ChatRequest)의 정보(query, 세션id)를 바탕으로 

2. 쓸 수 있는 도구들을 생성한다
3. LLM으로 Agent를 만든다
	```PYTHON
	def build_stateful_agent(llm, tools) -> AgentExecutor:
    """
    LLM과 도구(Tools)를 결합하여 기억력을 가진 에이전트를 만듭니다.
    """
    # 1. 시스템 프롬프트
    system_prompt = (
        "You are a helpful and accurate AI assistant for a travel planning service.\n\n"
        "**<response_format_guide>**\n"
        "When you get results from a tool (like `recommend_nearby_places`), that data contains technical fields (e.g., `x`, `y`, `id`).\n"
        # "When you get results from a tool (like `search_places`), that data contains technical fields (e.g., `x`, `y`, `id`).\n"
        "In your text response to the user, **NEVER** mention these technical fields.\n"
        "**ONLY** use human-readable information like `name`, `road_address`, `phone`, and `category` to create a natural summary.\n"
        "**</response_format_guide>**\n"
    )

    # chat_history: 이전 대화 내용을 넣을 공간
    # agent_scratchpad: AI가 도구를 사용한 생각의 과정을 적는 공간
    prompt = ChatPromptTemplate.from_messages(
        [
            ("system", system_prompt),
            MessagesPlaceholder(variable_name="chat_history", optional=True),
            ("system", "The user's session ID is: {session_id}"),
            ("human", "{input}"),
            MessagesPlaceholder(variable_name="agent_scratchpad"),
        ]
    )

    # 2. 에이전트 생성 (LLM + 도구 + 프롬프트)
    agent = create_tool_calling_agent(llm, tools, prompt)

    # 3. 실행기 생성
    agent_executor = AgentExecutor(
        agent=agent,
        tools=tools,
        verbose=True,
        return_intermediate_steps=True,
        max_iterations=15,
    )

    return agent_executor
	```
	
4. 에이전트는 `get_agent_response`함수로 필요한 도구를 사용하여 실행한다.
	```python
	def get_agent_response(agent, request: ChatRequest, history: list) -> ChatResponse:
    """
    사용자 쿼리와 세션 ID를 받아,
    대화형 응답과 구조화된 도구 데이터를 함께 반환
    """
    # agent.invoke()는 모든 실행 정보를 담은 dict를 반환
    result = agent.invoke(
        {
            "input": request.query,
            "chat_history": history,
            "session_id": request.session_id,
        },
    )

    print(result)

    # 4. 결과 파싱
    # 4-1. AI 답변 텍스트
    ai_message = remove_thinking_tags(result["output"])

    # 4-2. 도구 사용 기록 추출
    # steps 구조 : [(AgentAction, Observation), (AgentAction, Observation), ...]
    steps = result.get("intermediate_steps", [])

    tool_data_list = []

    for action, observation in steps:
        # observation이 보통 문자열로 되어있어 JSON이면 파싱해서 넣음
        parsed_output = safe_json_load(observation)

        # 매핑된 액션 리스트 가져오기 (없으면 빈 리스트)
        actions = TOOL_ACTION_MAP.get(action.tool, [])

        tool_data_list.append(
            ToolCallData(
                tool_name=action.tool,
                tool_output=parsed_output,
                frontend_actions=actions,
            )
        )

    # 3. API 엔드포인트에서 사용할 수 있도록 반환
    return ChatResponse(
        response=ai_message,
        tool_data=tool_data_list,
    )
	```


>[!QUESTION] intermediate_steps - 중간 단계 관리 
>- Agent의 메모리 역할
>- 지금까지 **Agent가 취한 모든 행동과 각 행동의 결과(observation)를 순서대로 저장** 
>- 매 단계마다 LLM에게 제공되어 "여태까지 무엇을 했고, 무엇을 알아냈는지"에 대한 컨텍스트를 제공

---
### 4.2.  정리 
- 세션별 히스토리 불러오기 (`get_session_history`) → **사실상 Memory 역할**
- `router_chain.invoke(...)`로 **Intent 분류 (CONVERSATION vs TOOL_USE)**
- Intent에 따라 LLM에게 줄 히스토리 전략 변경
    - 대화 모드: 전체 히스토리
    - 도구 모드: HumanMessage만
- 도구 생성 → Agent 조립 → Agent 실행
- 결과를 ChatResponse로 묶어서 반환 + 히스토리 업데이트


























