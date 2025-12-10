---
{"dg-publish":true,"permalink":"/프로젝트/나만무/AI-Agent/LangGrpah 실전 패턴/","noteIcon":"","created":"2025-12-03T14:53:06.723+09:00","updated":"2025-12-09T17:20:11.147+09:00"}
---



### 0.1.  목차

- [[#1.  흐름 - Router ➡ Agent ➡ Tools|1.  흐름 - Router ➡ Agent ➡ Tools]]
- [[#2.  사전 - 상태 정의 (feat. AgentState)|2.  사전 - 상태 정의 (feat. AgentState)]]
- [[#3.  Router Node - 대화 흐름의 시작점|3.  Router Node - 대화 흐름의 시작점]]
	- [[#3.  Router Node - 대화 흐름의 시작점#3.1.  Router Node 역할|3.1.  Router Node 역할]]
	- [[#3.  Router Node - 대화 흐름의 시작점#3.2.  코드 예시|3.2.  코드 예시]]
- [[#4.  Agent Node - 에이전트의 두뇌|4.  Agent Node - 에이전트의 두뇌]]
	- [[#4.  Agent Node - 에이전트의 두뇌#4.1.  역할|4.1.  역할]]
- [[#5.  ToolNode - 실제 작업 실행|5.  ToolNode - 실제 작업 실행]]
	- [[#5.  ToolNode - 실제 작업 실행#5.1.  올바른 ToolNode 패턴|5.1.  올바른 ToolNode 패턴]]
	- [[#5.  ToolNode - 실제 작업 실행#5.2.  툴 등록|5.2.  툴 등록]]
	- [[#5.  ToolNode - 실제 작업 실행#5.3.  tool_node 등록|5.3.  tool_node 등록]]
- [[#6.  Update State Node - 상태 업데이트 전용|6.  Update State Node - 상태 업데이트 전용]]
- [[#7.  Node/Edge 설계 패턴|7.  Node/Edge 설계 패턴]]
- [[#8.  완성 - agent_graph 생성 및 실행|8.  완성 - agent_graph 생성 및 실행]]
	- [[#8.  완성 - agent_graph 생성 및 실행#8.1.  실행|8.1.  실행]]



>[!tip] 핵심 패턴 
>LangGraph로 에이전트를 구성할 때 가장 강력한 접근은 **Node를 역할별로 분리하고, State 기반으로 흐름을 연결하는 패턴**

## 1.  흐름 - Router ➡ Agent ➡ Tools  


## 2.  사전 - 상태 정의 (feat. AgentState)

- LangGraph의 Node들은 서로 직접 데이터를 주고받지 않는다.
- 대신, State라는 하나의 거대한 공용 저장소를 바라봄 <br>![Pasted image 20251122141726.png](/img/user/supporter/image/Pasted%20image%2020251122141726.png)

>[!question] 상태 정의란❓
>- State에 **"어떤 칸(Key)이 있고, 각 칸에는 어떤 형태의 자료(Type)가 들어가야 하는지"** 규칙을 정하는 것
>- 아래와 같은 장점이 있다.
>	1. 데이터 통일 
>	2. 기억 유지 : 여러 step을 거치더라도 history가 사라지지 않고 끝까지 유지되도록 보장 
>	3. 타입 안정성 : 엉뚱한 타입의 데이터가 들어오는 것을 방지 
>

```python
# =========================
# 1. 상태 정의
# =========================
class AgentState(TypedDict, total=False):
    """LangGraph 상태 타입 분리 (순환 import 방지)"""

		# 대화 기록(⭐)
    messages: Annotated[Sequence[BaseMessage], operator.add]
    session_id: str
    # 라우터 결과 (제어 흐름)
    # 다음 단계로 무엇을 할지 결정하는 플래그 
    intent: (
        Literal[
            "NEW_SEARCH", "REFINEMENT", "CONVERSATION", "FOLLOW_UP", "REFINE_EXCLUDE"
        ]
        | None
    )
    # 각종 커스텀 속성들(장소 관련 기능에서 필요한 속성들)
    last_recommended_places: List[SimplePlace]
    excluded_place_ids: List[str]  # 제외할 장소 ID 리스트 (도구 호출 시 사용)
```
`TypeDict`❓
- 파이썬 `dict`에 타입 힌트를 강제하는 기술 
- 그래프에서 주고받는 공통 상태 스키마 
- `dict`와 달리 각 키 타입이 명시돼서 IDE/ 타입체커에 유리 (미리 경고)

`Annotated[list[BaseMessage], operator.add]`
- Langraph의 reducer 설정 
- `operator.add` : 새로운 메시지가 들어오면 기존 리스트를 지우지 말고, 기존 리스트 뒤에 append하라는 명령어

>[!QUESTION] LangGraph에서 Reducer란❓
>새로운 데이터가 들어왔을 때, **기존 상태(State)를 어떻게 업데이트할지 결정하는 규칙** 
>- 없다면❓(기본값) 새로운 값이 기존 값을 덮어씀 
>- 있다면❓지정된 규칙에 따라 기존값과 새로운 값을 합침 

`BaseMessage`
- *개념* : Langchain, Langgraph 생태계에서 사용되는 **모든 대화 메시지 객체들의 부모 클래스**
- *왜 쓰는가❓*LLM은 단순 글자뿐만 아니라 "누가 말했는지(Role)"도 구분이 必
- *주요 하위 클래스*
	- `SystemMessage`: AI의 성격 부여 (시스템 설정)
	- `HumanMessage`: 사용자 질문
	- `AIMessage`: AI의 답변
	- `ToolMessage`: 검색 도구, 계산기 등의 실행 결과값
- 즉, `chat_history: Annotated[list[BaseMessage], operator.add]`에서 `chat_history`변수는 사용자의 질문, AI 답변, 시스템 설정 등 어떤 종류의 메시지든 다 들어갈 수 있는 리스트라는 것 




## 3.  Router Node - 대화 흐름의 시작점

> Router Node : "**이번 입력이 무엇을 원하는지**" 를 판단하는 단계

main agent가 실행되기 전에 router_node를 통해 사용자의 의도 파악을 먼저 하곤 한다

### 3.1.  Router Node 역할

> [!tip] LangGraph에서 Router Node의 역할 
> 1. 현재 상태(state)를 받아서 
> 2. 최근 메시지를 필터링 
> 	- 몇 개 사용할건지 
> 	- 최근 메시지를 인간 메시지로 바꿀건지(Bedrock 제약 시)
> 3. LLM으로 의도를 분류
> 4. 의도 결과를 필터링된 state에 붙여
> 5. 다시 state로 돌려주는 함수 

### 3.2.  코드 예시 
```python

# =========================
# 라우터 체인 - Pydantic 모델을 LLM에 강제하는 라우터 체인
# =========================
router_chain = router_prompt | global_llm.with_structured_output(IntentClassifier)


# =========================
# 라우터 노드
# =========================
def router_node(state: AgentState) -> AgentState:
    messages = state.get("messages", [])
    # 최근 10개 메시지만 사용
    recent_messages = messages[-10:] if len(messages) > 10 else messages

    # Bedrock 제약: 대화는 항상 사용자 메시지로 시작해야 함
    first_human_idx = next(
        (idx for idx, msg in enumerate(recent_messages) if msg.type == "human"),
        None,
    )
    
    # 최근 메시지를 인간 message로 
    if first_human_idx is not None and first_human_idx > 0:
        recent_messages = recent_messages[first_human_idx:]
    elif first_human_idx is None:
        # 안전장치: 없으면 원본 messages에서 마지막 사용자 메시지만 사용
        last_human = next(
            (msg for msg in reversed(messages) if msg.type == "human"),
            None,
        )
        recent_messages = [last_human] if last_human else []

    # 의도 분류
    classification = router_chain.invoke({"messages": recent_messages})
    classification = IntentClassifier.model_validate(classification)
    
    # ✅ state에 intent값 세팅 
    return {
        "intent": classification.intent,
    }
```
 `invoke`는 아래의 실행 흐름을 가진다
	1. `router_prompt`가 `invoke` dict 인자를 받음
	2. 그 메시지가 `router_chain`에 들어감 
	3. LLM이 스키마에 맞는 JSON을 생성해서 반환 (여기서는 `IntentClassifier`로 정의됨)

## 4.  Agent Node - 에이전트의 두뇌 

LangGraph에서 가장 중요한 노드 중 하나
> 도구 "**요청**"을 만드는 노드(tool_calls를 생성) by LLM


### 4.1.  역할 

- 도구 실행 자체는 agent_node가 하지 않는다 ❗
- 사용자의 입력, 히스토리, scratchpad를 보고 **"지금 어떤 도구를 어떻게 호출해라"는 지시를 만듬⭐**
- 만약, 도구가 필요 없으면 최종 output 생성

>[!QUESTION] 그럼 tool 실행은 언제해❓
>실제 tool 실행은 ToolNode가 담당한다.

```scss
사용자입력
   ↓
agent_node (LLM → tool_calls 생성)
   ↓ (만약 tool_calls가 존재한다면)
tool_node (실제 도구 실행)
   ↓
agent_node (도구 결과를 보고 후속 판단)
   ↓
END (최종 응답)
```

- 에이전트의 두뇌 = `agent_node`
- 에이전트의 손과 발 = `ToolNode`




## 5.  ToolNode - 실제 작업 실행

AgentNode에서 **도구 호출이 결정되면 ToolNode에서 실제 기능이 실행**


### 5.1.  올바른 ToolNode 패턴 

1. 도구는 **“순수 함수”** 형태로 만들 것 ⭐
    - 상태를 들고 있지 않음
    - 입력 = 파라미터
    - 출력 = JSON or dict
        
2. 도구 내부에서 DB/캐시/네트워크 작업은 가능  
	- 하지만 “State를 수정”하지 말 것  
	- **상태 업데이트는 `update_state_node`에서만 ⭐**



agent_node가 어떠한 툴이 필요한지 알려주면 tool_node가 실제 도구를 실행한다.

1. 툴 등록
2. ToolNode 등록 

### 5.2.  툴 등록 

`@tool`을 써서 메서드를 tool로 등록하고 list로 모아놓으면 됨 

```python
def get_workspace_tools():
		....
		
    @tool
    def recommend_places_by_all_users(workspace_id: str):
	    """
		  도구 설명란 
	    """
	    
    return [recommend_places_by_all_users] 
```

```python
def create_nest_tools():
    # 모든 도구 리스트 합치기
    all_tools = []

    # 리스트 더하기
    all_tools.extend(get_workspace_tools())
    all_tools.extend(get_place_tools())
    all_tools.extend(get_poi_tools())

    return all_tools
```

### 5.3.  tool_node 등록

```python
    tools = create_nest_tools()
    # 툴 Node 형태로 변환 후 graph에 등록 
    tool_node = ToolNode(tools)
    workflow.add_node("tools", tool_node)
```



## 6.  Update State Node - 상태 업데이트 전용 

 LangGraph 설계 철학에서 중요한 부분은 **도구는 순수하게 두고, State 업데이트는 별도 노드에서 한다.**

>[!QUESTION] 상태 업데이트 노드를 따로 두는 이유
>1. 도구 재사용성 증가
>2. 상태 변경이 명확
>3. 디버깅 쉬움


## 7.  Node/Edge 설계 패턴

```python
workflow.set_entry_point("router")

workflow.add_conditional_edges(
    "router",
    route_by_intent,
    {"agent": "agent", "refine_exclude": "refine_exclude"},
)

workflow.add_conditional_edges(
    "agent",
    should_continue,
    {"tools": "tools", END: END},
)

workflow.add_edge("tools", "update_state")
workflow.add_edge("update_state", "agent")
workflow.add_edge("refine_exclude", "agent")
```


## 8.  완성 - agent_graph 생성 및 실행
```python

# =========================
# 4. 그래프 구성
# =========================
def create_agent_graph():
    """
    LangGraph 기반 에이전트 그래프 생성
    """
    # StateGraph 초기화
    workflow = StateGraph(AgentState)

    # 도구 노드 생성
    tools = create_nest_tools()
    tool_node = ToolNode(tools)

    # 노드 추가
    workflow.add_node("router", router_node)
    workflow.add_node("agent", agent_node)
    workflow.add_node("tools", tool_node)

    # 엣지 정의
    # 시작 -> 라우터
    workflow.set_entry_point("router")

    # 라우터 -> 에이전트 (항상)
    workflow.add_edge("router", "agent")

    # 에이전트 -> 조건부 분기 (도구 호출 or 종료)
    workflow.add_conditional_edges(
        "agent", should_continue, {"tools": "tools", END: END}
    )

    # 도구 -> 에이전트 (도구 실행 후 다시 에이전트로)
    workflow.add_edge("tools", "agent")

    # 그래프 컴파일
    return workflow.compile()


# 전역 그래프 인스턴스 (싱글톤)
agent_graph = create_agent_graph()
```



### 8.1.  실행 


> StateGraph는 "상태 딕셔너리"를 계속 수정하면서 마지막에 전체 state를 반환