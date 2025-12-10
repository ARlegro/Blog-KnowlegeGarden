---
{"dg-publish":true,"permalink":"/프로젝트/나만무/AI-Agent/LangGraph 핵심 기술 (state + checkpointer)/","noteIcon":"","created":"2025-12-03T14:53:06.843+09:00","updated":"2025-12-10T14:01:41.666+09:00"}
---

### 0.1.  목차

- [1.  State - 에이전트의 공용 저장소 ⭐⭐](#1--state---%EC%97%90%EC%9D%B4%EC%A0%84%ED%8A%B8%EC%9D%98-%EA%B3%B5%EC%9A%A9-%EC%A0%80%EC%9E%A5%EC%86%8C-)
	- [1.1.  기본 개념](#11--%EA%B8%B0%EB%B3%B8-%EA%B0%9C%EB%85%90)
	- [1.2.  예시](#12--%EC%98%88%EC%8B%9C)
	- [1.3.  State 특징](#13--state-%ED%8A%B9%EC%A7%95)
	- [1.4.  그래프 생성 시 State 정보](#14--%EA%B7%B8%EB%9E%98%ED%94%84-%EC%83%9D%EC%84%B1-%EC%8B%9C-state-%EC%A0%95%EB%B3%B4)
- [2.  번외 : state 정의하는 여러가지 방법](#2--%EB%B2%88%EC%99%B8--state-%EC%A0%95%EC%9D%98%ED%95%98%EB%8A%94-%EC%97%AC%EB%9F%AC%EA%B0%80%EC%A7%80-%EB%B0%A9%EB%B2%95)
	- [2.1.  MessagState - 기본형 상태](#21--messagstate---%EA%B8%B0%EB%B3%B8%ED%98%95-%EC%83%81%ED%83%9C)
	- [2.2.  직접 정의 state](#22--%EC%A7%81%EC%A0%91-%EC%A0%95%EC%9D%98-state)
- [3.  Reducer - State 업데이트 규칙](#3--reducer---state-%EC%97%85%EB%8D%B0%EC%9D%B4%ED%8A%B8-%EA%B7%9C%EC%B9%99)
- [4.  Checkpointer - 대화의 영속적 기억 장치](#4--checkpointer---%EB%8C%80%ED%99%94%EC%9D%98-%EC%98%81%EC%86%8D%EC%A0%81-%EA%B8%B0%EC%96%B5-%EC%9E%A5%EC%B9%98)
- [5.  Thread ID - 세션을 구분하는 메타데이터](#5--thread-id---%EC%84%B8%EC%85%98%EC%9D%84-%EA%B5%AC%EB%B6%84%ED%95%98%EB%8A%94-%EB%A9%94%ED%83%80%EB%8D%B0%EC%9D%B4%ED%84%B0)
- [6.  흐름 정리 : State + Checkpointer + Node 흐름 전체](#6--%ED%9D%90%EB%A6%84-%EC%A0%95%EB%A6%AC--state--checkpointer--node-%ED%9D%90%EB%A6%84-%EC%A0%84%EC%B2%B4)
	- [6.1.  사용자가 메시지를 보낸다](#61--%EC%82%AC%EC%9A%A9%EC%9E%90%EA%B0%80-%EB%A9%94%EC%8B%9C%EC%A7%80%EB%A5%BC-%EB%B3%B4%EB%82%B8%EB%8B%A4)
	- [6.2.  LangGraph가 thread_id 기반으로 체크포인트 조회](#62--langgraph%EA%B0%80-thread_id-%EA%B8%B0%EB%B0%98%EC%9C%BC%EB%A1%9C-%EC%B2%B4%ED%81%AC%ED%8F%AC%EC%9D%B8%ED%8A%B8-%EC%A1%B0%ED%9A%8C)
	- [6.3.  그래프 실행](#63--%EA%B7%B8%EB%9E%98%ED%94%84-%EC%8B%A4%ED%96%89)
	- [6.4.  각 노드가 끝날 때마다 상태가 Checkpointer에 자동 저장](#64--%EA%B0%81-%EB%85%B8%EB%93%9C%EA%B0%80-%EB%81%9D%EB%82%A0-%EB%95%8C%EB%A7%88%EB%8B%A4-%EC%83%81%ED%83%9C%EA%B0%80-checkpointer%EC%97%90-%EC%9E%90%EB%8F%99-%EC%A0%80%EC%9E%A5)
	- [6.5.  다음 요청이 오면 thread_id로 이전 상태부터 다시 시작](#65--%EB%8B%A4%EC%9D%8C-%EC%9A%94%EC%B2%AD%EC%9D%B4-%EC%98%A4%EB%A9%B4-thread_id%EB%A1%9C-%EC%9D%B4%EC%A0%84-%EC%83%81%ED%83%9C%EB%B6%80%ED%84%B0-%EB%8B%A4%EC%8B%9C-%EC%8B%9C%EC%9E%91)


> Langgraph의 가장 강력한 부분은 state를 중심으로 Agent를 구동한다는 점 

이것을 할 수 있게 하는 내부 구조들을 공부할 것 
1. state - 노드 간 공유되는 공용 저장소 
2. Reducer - 상태 업데이트 규칙 
3. Checkpointer - 상태 영속화(세션 기억 장치)
4. Thread ID - 세션 구분 메타데이터
5. State + Graph + Checkpointer 전체 실행 흐름 

---
## 1.  State - 에이전트의 공용 저장소 ⭐⭐


---
### 1.1.  기본 개념 
> **모든 Node가 읽고/쓰고 공유하는 "전역 상태 딕셔너리"**

LangGraph에서 Node들은 서로 직접 데이터를 넘기지 않는다.<br>
모두 `State`를 바라보고, 필요할 때 `State`를 수정한다.
- LangGraph 워크플로우는 여러 Node로 구성되는데,  
- 이 Node들은 각각 독립적인 역할을 가지면서도 같은 State를 바라본다.  
- 즉, **State는 그래프 전체의 “공용 저장소” 역할**을 한다.

---
### 1.2.  예시 
```python
class AgentState(TypedDict, total=False):
    messages: Annotated[Sequence[BaseMessage], operator.add]  # 메시지 히스토리
    session_id: str  # 유저 세션
    intent: Literal["NEW_SEARCH", "REFINEMENT", "CONVERSATION", "FOLLOW_UP", "REFINE_EXCLUDE"] | None
    last_recommended_places: List[SimplePlace]   # 추천 결과
    excluded_place_ids: List[str]               # 제외할 장소 리스트
```
State는 일반적인 dict처럼 보이지만,  
LangGraph는 이 State를 기반으로 **상태 머신(state machine)** 형태로 에이전트를 실행한다.

---
### 1.3.  State 특징
1. *구조화된 정보* - 구조적·기능적 메모리
	- Agent의 목표, 사용 도구들의 특징 등 **Workflow에 필요한 모든 데이터들을 구조화하여 저장**할 수 있다.
	- 즉, 단순히 메시지들을 저장하는게 아니라 intent, 추천 목록, 제외 목록 등 
2. *모든 Node가 공유*
	- 모두 같은 State 객체를 읽고 수정한다.
	- 이 방식 덕분에 여러 단계의 도구 호출,재추천, 제외·필터 작업 같은 multi-step 로직이 안정적으로 동작
	  
3. *흐름 제어를 가능하게 해줌* - 그래프 전체의 제어 신호 역할
	- 단순 저장소가 아니라 **그래프의 다음 흐름을 결정하는 조건 판단 기준**이 되기도 한다.
	- 그래프의 다음 Node결정하는 조건부 Edge는 state의 현재 값을 기반으로 함
	```python
	# router -> agent or refine_exclude (의도에 따라 분기)
    workflow.add_conditional_edges(
        "router",
        route_by_intent,  # 이게 노드
        {"agent": "agent", "refine_exclude": "refine_exclude"},
    )
	```
	- router_node는 State.intent를 설정하고
	- Edge는 intent 값을 보고 다음 Node를 선택

---
### 1.4.  그래프 생성 시 State 정보 
```python
def create_agent_graph():
    """
    LangGraph 생성
	  """	
    workflow = StateGraph(AgentState)
```
Graph는 이 지정된 State 타입에 따라:
- State 구조를 관리하고
- Reducer 규칙을 적용하며
- Node 간 데이터 흐름을 조정한다.

---
## 2.  번외 : state 정의하는 여러가지 방법 
> State 정의 : Graph의 노드와 노드 간 공유하는 상태를 정의하는 것 


---
### 2.1.  MessagState - 기본형 상태 
```python
from langgraph.graph import StateGraph, MessagesState
graph = StateGraph(MessagesState)
```
- 개발자들의 편의를 위해 미리 만들어 둔 기본형 상태 
- 실제로 `MessageState`는 아래처럼 정의되어 있다.
	```python
	class MessagesState(TypedDict):
	    messages: Annotated[list[AnyMessage], add_messages]
	```
- 메시지만 관리하면 되는 기본적인 챗봇 만들 때 코드 줄여줌 

---
### 2.2.  직접 정의 state 

*TypeDict 형태* 
```python
from typing import Annotated, TypedDict
from langgraph.graph.message import add_messages

# 사용자가 직접 정의하는 상태명
class GraphState(TypedDict):
    # 메시지가 덮어씌어지지 않도록 쌓이도록 add_messages 설정 필
    messages: Annotated[Sequence[BaseMessage], operator.add]
```
- 채팅 기록을 직접 관리할 수 있다.
- 일반적으로 `TypedDict`형식을 사용함 
- `add_messages`라는 `reducer` 함수 필요
	- `reducer` : "이전 상태 + 새로운 입력" ➡ "새로운 상태"를 만드는 함수 
		- 즉, state를 어떻게 업데이트할지 규칙을 정의하는 함수 

기본형 상속 
```python
from langgraph.graph import MessagesState

# MessagesState를 상속받음 -> messages 기능은 자동으로 포함됨
class MyCustomState(MessagesState):
    user_name: str    # 내가 추가한 변수
    turn_count: int   # 내가 추가한 변수
# 그래프 생성 시 확장된 상태 사용 
graph = StateGraph(MyCustomState)    
```
- 위처럼, LangChain이 제공하는 messageState를 상속하여 새로운 변수를 만들 수 있다.

---
## 3.  Reducer - State 업데이트 규칙 
> State가 어떻게 덮어씌어지고 쌓이는지 정의하는 것은 Reducer로 제어가 가능하다.

>[!QUESTION] Reducer란?
>새 입력이 들어왔을 때  
>**기존 상태 + 새 값 → 어떤 값이 최종 상태가 되는지** 결정하는 함수.
>```python
>messages: Annotated[Sequence[BaseMessage], operator.add]
>```
>- operator.add 
>	- 기존 message 리스트 + 새 메시지 리스트
>	- 즉, 기존것과 합쳐서 append하라는 명령어 

*왜 필요한가❓*
- 어떤 필드는 이전 기록이 쌓여야 할 수도 있고, 어떤 필드는 덮어쓰기(교체)가 돼야 할 수도 있다. 이런 것들을 **제어**하는 것이 "Reducer"이다.
- LangChain에는 이런 개념이 없어서 어떤 값이 어덯게 업데이트되어야 하는지는 프롬프트/체인 로직으로 강제해야한다(하드코딩)

---
## 4.  Checkpointer - 대화의 영속적 기억 장치

[[프로젝트/나만무/AI-Agent/LangGraph - Checkpointer 도입\|LangGraph - Checkpointer 도입]]

> `State`객체의 내용을 메모리/DB에 "스냅샷"형태로 자동 저장하는 장치

- LangGraph에는 State를 영구적으로 저장하고 다음 호출에서 그대로 불러오는 **Checkpointer**가 내장돼 있다.
- Checkpointer 덕분에 Agent는 세션이 종료되거나 오류가 발생해도 **중단 지점부터 작업 재개 가능⭐**

```python
from langgraph.graph import StateGraph
from langgraph.checkpoint.memory import MemorySaver

builder = StateGraph(...)
# ... define the graph
# checkpointer = # postgres
checkpointer = MemorySaver() # 인메모리 버전
```

*✔Checkpointer가 하는일* 
1. *Graph 실행 전*
	- thread_id 기반으로 **“이전에 저장된 state”가 있는지 조회**
	- 있다면 그 state를 불러와 이번 실행에 합침
	  
2. *그래프 실행 중*
	- 각 노드가 끝날 때 상태를 스냅샷으로 저장

3. *그래프 실행 후*
	- 최종 상태도 저장해둠


✔*다양한 종류*
- DB는 다양한 종류 지원한다
- ex. 인메모리(`MemorySaver`), `PostgresSaver`, `RedisSaver` 등 

---
## 5.  Thread ID - 세션을 구분하는 메타데이터
- 사용자는 **대화를 새로 시작할 때 특정 ThreadID(or SessionId)를 기반으로 과거의 모든 정보를 즉시 복원할 수** 있다.

> [!WARNING] State와 Thread_id를 구분하기 
> - State : 그래프 내부에서 다루는 데이터
> - Thread ID : Checkpointer가 상태를 구분하기 위한 외부 메타데이터 

설정 법
```python
# 2. LangGraph 실행 (체크포인터 사용)
# thread_id를 session_id로 사용하여 세션별 상태 유지
config = {
    "configurable": {
        "thread_id": request.session_id
    }
}

final_state = await agent_graph.ainvoke(initial_state, config)
```
- config = 그래프가 실행전 `config`안의 정보를 읽어서 ("thread_id", 값)조합으로 세션키를 만든것
	- `config.configurable.thread_id` => **이게 session 키**
	- 이 키로 이전에 저장된 state가 있는지 체크 
- 그 안의 configurable은 메타정보 컨테이너 역할을 한다.
- `config`는 단순 메타정보라 `initial_state`필드값과 무관한다.❗❗
- session 키에 해당하는 과거 state + intial_state를 합쳐서 새로운 state로 Graph를 이어감
- *복원되는 정보*
	- *대화 State*
	- *메시지 기록* : 이전 LLM 호출 기록, 사용자 입력 등 전체 히스토리
	- *도구 실행 결과* : 과거에 실행된 도구의 출력 값들 
	- *사용자 컨텍스트* : 대화 초기에 제공했던 배경 등 
> [!INFO] ⭐정리 
> - thread_id는 "LangGraph" 내부에서 상태 스냅샷을 관리하는 Key
> - config는 LLM이 읽지 않는 별도의 채널이다.



---
## 6.  흐름 정리 : State + Checkpointer + Node 흐름 전체

위와 같은 기능들은 LangGraph에서 실행할 때 아래 순서로 돌아간다.

---
### 6.1.  사용자가 메시지를 보낸다
```python
initial_state = {"messages": [HumanMessage(...)]} config = {"configurable": {"thread_id": "user-123"}}
```

---
### 6.2.  LangGraph가 thread_id 기반으로 체크포인트 조회

- “user-123” 상태가 **저장돼 있으면 불러온다**
- **initial_state와 merge**
- 이전 값과 merge할 때 필드의 **reducer 값에 따라 다르게 저장**된다.
	- ex. messages는 append, intent는 덮어쓰기, excluded_place_ids는 누적 or 덮어쓰기 선택
        
---
### 6.3.  그래프 실행

아래는 나만무에서 구현한 흐름 예시 
1. router_node → intent 분석
2. agent_node → 도구 필요 여부 판단
3. tool_node → 실제 도구 실행
4. update_state_node → 결과를 State에 반영
5. 다시 agent_node → 반복 판단
6. END 도달 시 종료


---
### 6.4.  각 노드가 끝날 때마다 상태가 Checkpointer에 자동 저장

아래는 나만무에서 구현한 state에 저장하는 값들 예시 
- messages
- intent
- 추천 결과
- 제외 ID
- 기타 모든 State 필드
    
---
### 6.5.  다음 요청이 오면 thread_id로 이전 상태부터 다시 시작

부제목 그대로이다. 
- 이게 LangGraph가 **장기 대화 / multi-step reasoning / 반복 도구 호출 / 재시도 / 복구**에 강한 근본 이유
- state + checkpointer 구조는 `LangChain`의 단순한 메모리 관리 한계를 뛰어넘음
- 견고하고 상태를 인지하는 `stateful`한 장기 실행 에이전트 시스템을 구축할 수 있게 함 


