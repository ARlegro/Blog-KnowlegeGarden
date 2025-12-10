---
{"dg-publish":true,"permalink":"/프로젝트/나만무/AI-Agent/LangGraph - Checkpointer 도입/","noteIcon":"","created":"2025-12-03T14:53:06.830+09:00","updated":"2025-12-09T17:20:11.093+09:00"}
---



## 1.  기존에 있던 문제 

기존 : `@tool`로 등록한 도구와 프롬프트를 활용해서 이전 기억들을 활용하려 시도
변경 : LangGraph 노드 설계 및 Checkpointer 사용 


## 2.  Langgraph 권장 패턴 
별도의 '추천 로직 노드'를 사용
- 도구는 가능한 한 순수한 함수(상태 비저장)로 유지하고, 
- **상태 업데이트 로직은 `StateGraph`의 노드 함수 안에서** 처리하는 것이 LangGraph의 설계 철학
	- `"~말고 다른걸로 추천해줘"` 같은 요청은 사용자의 의도를 파악하는 **라우팅 노드(조건부 엣지)** 를 통해 처리
	- 상태를 업데이트하는 별도의 노드로 보낸 후, 추천 도구를 사용하는 노드로 이동시킵니다.

이렇게 하면 장점 
- 워크플로우가 명확해지고 
- 상태 관리가 중앙 집중화되어 디버깅과 유지보수가 용이

## 3.  CheckPointer

> LLM애플리케이션이 실행되는 동안 상태(State)를 저장하여 나중에 그 시점부터 다시 시작하거나 내용을 수정할 수 있게 해주는 기능



*💢삽질*
- 이 기능은 LangGraph에서 제공하는 기능이다. 
- 지금까지 LangChain만 쓰다보니 이 기능이 있는지도 몰라서 상태를 어떻게 보존하고 조작할 수 있을까❓에 대해서 삽질을 많이 했던 것 같다.(기존)
- Langchain같은 경우 단순히 history를 저장하는 것인데 그냥 이걸로 이전 

### 3.1.  💢Langchain에서 AI-Agent 챗봇이용시 어려웠던 점 
- Langchain에서는 상태에 대한 관리가 힘들다
- 가령, 유저가 AI-Agent에게 어떤 경유지들을 포함해서 각 장소들을 추천받았다고 가정하자. 만약, 사용자가 특정 장소가 맘에 안들어서 "~맘에 안드는데 이거 말고 다른거 추천해줘"라고 했다고 하자. 이 경우 대체되려는 장소의 근처 장소를 추천해야하고 기존의 장소들은 유지하고 그 모든 추천장소들과 겹치지 않는 새로운 장소를 추천해줘야 한다. 
- LangChain의 `@tool` 등록 + 프롬프트 + session_history 이것만으로는 계쏙해서 오류가 발생했다.
- 관련 DB를 둘까도 했는데 너무 비효율적으로 보였고 확장성이 부족해보였다.


### 3.2.  CheckPointer란❓
#세이브파일

> 에이전트 그래프가 어떤 상태(State)까지 왔는지 스냅샷으로 저장해두는 저장소

내가 자주 즐겨봤던 웹툰 중에 "체크포인트"라고 있었다. 이는 원하면 특정 시간으로 되돌아가는 건데, 이걸 생각하면 이해하고 외우기가 쉬어진다.

체크포인터는 **각 단계에서의 상태를 기록**해서,
- 중간에 실패해도 **다시 이어서 실행**하거나
- `thread_id`(=session_id) 기준으로 **이전 대화 상태를 복원**할 수 있게 만들어준다.

Checkpointer는 대화의 맥락을 유지하고, 중간 개입이 가능하며 과거의 결과를 수정할 수 있는 지능형 에이전트 기능을 제공한다.


>[!tip] 이게 없다면 매번 AI-Agent 챗을 실행할때마다 새로운 상태로 시작해야 한다.
>Checkpointer는 대화의 맥락을 유지하고, 개입을 


### 3.3.  기본 사용법 

*1‍⃣ 생성 및 등록*
```python
from langgraph.graph import StateGraph
from langgraph.checkpoint.memory import MemorySaver

# 1. 메모리 기반 체크포인터 생성 
memory = MemorySaver()

# 2. 그래프 정의 (예시)
builer = StateGraph(...)
# 노드, 엣지 추가 

# 3. 그래프 컴파일 시 checkpointer 등록 
graph = builder.compile(checkpointer=memory)
```

*2‍⃣ 실행 시 설정* ⭐
- 체크포인트를 등록한다고 자동으로 되는게 아니다
- graph를 사용할 때 thread_id를 Config에 포함시켜 보내면 이 ID를 토대로 대화 세션을 구분하며 작업을 실행한다.
  
```python
# 1. 설정 만들기 : "이 작업은 session_id라는 이름의 세이브 파일을 써라"
config: RunnableConfig = {"configurable": {"thread_id": session_id}}


loop = asyncio.get_running_loop()
# 2. 실행하기 : 입력값(initail_state)과 설정(config)을 함께 전달
final_state =
		await loop.run_in_executor(None, partial(agent_graph.invoke, inialt_state, config))
```
- `graph.invoke`는 2개의 인자를 받을 수 있게 되어있다.
	1. `input` : 그래프에 넣을 재료 
	2. `config` : 실행 옵션 
- invoke함수 실행 시 Langgraph 내부는 `config`를 먼저 확인하여 `thread_id`에 해당하는 **이전 대화 기록인 `Checkpointer`를 불러와서 합친 다음에 실행**한다 ⭐


> thread_id별로 그래프 상태의 스냅샷을 저장



### 3.4.  어떻게 자동 저장되는가❓

- 만약, `memory = MemorySaver();`로 설정을 했다면 상태(State)들은 **LangGraph 내부 메모리에 저장**되며 외부에서 직접 보이지 않고 디스크에도 없다. 
- **LangChain 메시지 객체들이 자동으로 저장**되며 추후에 `MemorySaver`가 그 상태를 세션별로 저장·복원한다.


Chekcpointer는 에이전트 그래프가 어느 노드까지 실행되었는지, 그때의 상태가 무엇이었는지를 스냅샷 형태로 저장한다.


### 3.5.  Checkpointer 종류

이것도 기억이다보니 어딘가에 저장해야 한다.
Langgraph는 다양한 백엔드를 지원한다.

1. *인메모리 - `MemorySaver`*
	- 프로그램 종료 시 데이터 휘발
	- 개발, 테스트, 간단한 데모 시에 유용
	  
2. *`PostgreSQL` - `PostgresSaver`*
	- 강력한 관계형 DB
	- 실제 서비스에서 많이 쓰임 
3. *`Redis` - `RedisSaver`*
	- 고성능 인메모리 Key-value 저장소 


### 3.6.  필요 없어진 코드들

1. `session_history` 재사용 및 병합
	```python
	  # chat_v2.py Line 121-133
	  session_history = get_session_history(request.session_id)
	  chat_history = list(session_history.messages)  # <<< 외부 히스토리 로드
	
	  user_message = HumanMessage(content=request.query)
	  initial_messages = chat_history + [user_message]  # <<< 병합
	
	  initial_state: AgentState = {
	      "messages": initial_messages,  # <<< 체크포인터와 중복!
	      ...
	  }
	```
	- 체크포인터는 `AgentState.messages`에 그래프 내부에서 생성된 모든 메시지(ToolMessage 포함)를 저장하지만
	- session_history.messages를 불러올 때는 체크포인터에 접근하지 않는다.
	- Checkpointer는 모든 상태를 자동으로 관리하므로 코드가 매우 단순할 것 
```python
	user_message = HumanMessage(content=request.query)
	initial_state: AgentState = {
			"messages": [user_message],
			"session_id": request.session_id,
			"intent": None,
	}
```


> 대화 히스토리는 체크포인터가 자동으로 관리

2. `session_history` 추가 
```python
@router.post("", response_model=ChatResponse)
async def ask_agent_langgraph(request: ChatRequest) -> ChatResponse:

	
        # 6. 대화 히스토리 저장 💢
        session_history.add_user_message(request.query)
        session_history.add_ai_message(output)
```

- LangGraph에서는 `Checkpointer`가 **그래프 상태의 스냅샷을 자동으로 저장**한다.
- `thread_id`를 통해 `agent_graph`가 실행될 때, Checkpointer를 사용하는 한 그래프가 마지막 실행 상태에 자동으로 액세스하므로, 메시지를 `session_history`에 다시 추가하는 것은 불필요한 중복 저장입니다.(이는 Langchain에서나 했던 것)
- *권장 패턴*
	- `Checkpointer`로 최신 상태를 유지하고 
	- 별도의 ChatHistory 테이블에서만 사용자에게 표시할 메시지를 저장하는 것


![Pasted image 20251125034627.png](/img/user/supporter/image/Pasted%20image%2020251125034627.png)


