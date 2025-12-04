---
{"dg-publish":true,"permalink":"/나만무/AI Agent 아키텍쳐 리팩토링/","noteIcon":"","created":"2025-12-01T21:59:49.957+09:00","updated":"2025-12-05T00:25:21.976+09:00"}
---




## 1.  최종 아키텍쳐 Preview 
![Pasted image 20251205002510.png](/img/user/supporter/image/Pasted%20image%2020251205002510.png)


## 2.  배경 및 문제 인식 

초기 프로젝트는 기획 지연과 급박한 AI Agent 기능 도입으로 인해 코드 품질보다는 빠른 개발에 몰두하게됐다. 그 결과, 초반에는 “작동하는 것”에만 집중할 수밖에 없었고 구조적 완성도는 자연스럽게 뒤로 밀렸다.

AI Agent를 만들어보는 것은 처음이라 어떤 계층 어떤 패턴이 적절한지에 대한 기준이 부족했다. 공식 문서([LangChain, LangGraph](https://www.langchain.com/))에서도 아키텍처를 어떻게 설계하거나 계층화하라는 가이드를 보지 못해 시행착오가 많았다.

가장 맘에 안 들었던 코드는 "툴(Tool) 구조의 일관성 부족이었다.
Tool마다 반환 형식이 다르고, 처리 과정 또한 제각각이어서 다음과 같은 문제가 반복해서 발생했다.
기존 툴의 문제
```python

@tools 
def ~~ 
# ToolResult형식으로 반환하는 예시 
```
이렇게 만들고 설계한 AI Agent에는 아래와 같은 문제가 있다고 생각한다 
- *지나친 의존성*
	-  **순수 함수(Pure Function)**로서의 역할을 상실** : ToolResult 반환 형식에 지나치게 의존적인 비즈니스 로직 
	- 순수 함수 형태를 유지 못하고 곳곳의 코드에서 LangGraph 종속성 노출 
- *하나의 노드(Update_state_node)에서 모든 툴을 처리* ➡ **if-elif 지옥** 

>[!QUESTION] 들었던 의문 
>“툴로 등록한 함수는 정말 AI 전용이어야 할까?  
>아니면 어디서든 호출 가능한 순수 비즈니스 로직이어야 하지 않을까?”

이런 문제 인식과 의문이 프로젝트가 다 끝난 마다에도 리팩토링을 하고 싶은 마음을 불태웠다.

일단 내가 가장 이상적이라 생각하는 그림은 
1. Service는 순수한 도메인 로직만 담당하게 하고 
2. Tool은 Service를 호출하고 일관된 반환타입으로 감싸주는 역할 
3. Node는 Tool실행 결과를 받아 상태를 업데이트하는 단위로 생각했다.

이렇게 한다면 각 기능이 독립적이면서 확장이 가능한 구조라고 생각했다.

이를 달성하기 위해 스프링에서 AOP나 어댑터 패턴을 적용하던 경험이 떠올랐다.  
LangGraph에서도 동일한 방식의 계층 분리와 후처리 라우팅 구조를 만들 수 있을 것 같았다.

따라서, 기존의 무작위적인 툴, graph, 함수 구조를 걷어내고 새로운 패턴 구조를 도입하게 되었다. 


## 3.  최종 아키텍쳐 그림 및 핵심 원칙 (Remind)

![Pasted image 20251205002424.png](/img/user/supporter/image/Pasted%20image%2020251205002424.png)
1. **Service는 순수 비즈니스 로직** - LangChain/LangGraph에 대해 전혀 모름
2. **Tool은 얇은 어댑터** - Service 호출 + ToolResult 포장만 담당
3. **Node는 상태 관리** - Tool 결과를 받아서 State 업데이트


## 4.  계층별 책임

MateTrip AI Agent를 이루는 3계층 구조를 어떻게 설계했고 분리했는지에 대한 설명
**목적 : 도구-비즈니스로직-상태관리분리 등을 위해 설계**되었다

### 4.1.  Layer 1: Service (순수 비즈니스 로직)
> Service 계층은 프로젝트 전체에서 가장 중요한 원칙인 **“순수성(Purity)”**을 기반으로 설계

- LangChain, LangGraph와 완전히 독립적이다.⭐⭐
- ToolResult같은 반환형식이나 LangChain Message를 다루지 않는다.
- **DB 조회 → 도메인 로직 처리 → DTO 반환**이라는 가장 기본적인 책임만 가진다
- **재사용성**: REST API, CLI, 테스트 코드 등 어디서든 사용 가능

즉, Service 레이어는 "AI 기능을 위한 코드"가 아니라 프로젝트 전체의 비즈니스 로직을 담당하는 레이어로 설계해야 한다.

아래 예시는 DTO기반의 요청-응답 구조 
```python
class PlaceService:
    def find_replacement_places(
        self, request: ReplacePlaceRequest
    ) -> List[NearbyPlaceResponse]:
        """
        순수 비즈니스 로직 - ToolResult 모름

        ReplacePlaceRequest(6개 필드)
        - latitude, longitude, replace_count, excluded_place_ids,
          category, radius_km
        -> ReplacePlaceRequest DTO
        """
        places = self.repository.find_places_within_radius(
            latitude=request.latitude,
            longitude=request.longitude,
            radius_km=request.radius_km,
            category=request.category,
            limit=request.replace_count,
            excluded_place_ids=request.excluded_place_ids,
        )
        return [NearbyPlaceResponse.from_entity(p) for p in places]
```

핵심은 Service는 LangGraph의 존재를 모른다는점 
이 덕분에, Service는 유지보수성, 가독성, 테스트가능성, 재사용성이 모두 확보된다.


### 4.2.  Layer 2: Tool (LLM 어댑터)

> **얇은 래퍼 클래스**: 비즈니스 로직 없음, 변환만 수행

Tool은 LangGraph가 호출하는 함수이다. 따라서 LangGraph와 내부 서비스를 연계하는 중간 다리 역할을 담당한다.

여기에는 비즈니스 로직이 들어가면 안되고 아래의 3가지 역할을 위주로 하도록 설계했다.
1. LLM으로부터 전달받은 파라미터 검증
2. Service 호출을 위한 DTO 정규화
3. Service 결과 정규화 및 메타데이터 추가 

예시 
```python
@tool
def replace_places(replace_target_ids, latitude, longitude, ...):
    """LLM 어댑터 - Service 호출 + ToolResult 포장"""

    # 1) LLM에서 들어온 파라미터를 DTO로 캡슐화 
    request = ReplacePlaceRequest.create(
        latitude=latitude,
        longitude=longitude,
        replace_count=len(replace_target_ids),
        excluded_place_ids=excluded_place_ids,
        category=mapped_category,
        radius_km=radius_km,
    )

    # Service Layer 호출 (순수 비즈니스 로직)
    place_responses = PlaceService(db).find_replacement_places(request)

    # ToolResult로 감싸기 (일관된 JSON 스키마)
    return ToolResult(
        success=True,
        data=PlaceRecommendationData(
            places=[p.model_dump() for p in place_responses],
            replaced_place_ids=replace_target_ids,  # 메타정보
        ),
        # 메타 정보들 SUCCESS, replace_id, error_code 등 공통 필드 확장 
    ).model_dump()
```
Tool은 진짜 로직은 없고 단순 전달 및 결과를 변환해주는 어뎁터 역할

장점
- Service 로직 오염 안시킬 수 있다.
- Service 결과 표준화 가능 ➡ LLM이 이해하기 좋은 일관된 형태의 반환값 유지 가능
- Tool은 매우 얇아서 확장/변경이 쉬움


### 4.3.  Layer 3: Node (상태 관리)
> Tool 실행 결과를 기반으로 상태를 업데이트하는 계층 

Tool이 “데이터를 가져오는 역할”이라면 Node는 “그 데이터를 AgentState에 반영하는 역할”을 한다.

LangGraph에서는 ~~ 하면서 상태를 변경한다. AgentState변경하면서 

Node Layer의 역할
- `ToolResult`를 파싱하고
- `AgentState`에 반영하고
- 다시 Agent 노드로 돌아가는 흐름을 만든다

**파일 구조**:
```
app/agent/nodes/
├── handle_replace_places_node.py       # replace_places 전용
├── handle_place_recommendation_node.py # 일반 장소 추천 전용(여럿)
├── handle_travel_route_node.py         # create_travel_route 전용
└── ...
```

**예시**:
```python
def handle_replace_places_node(state: AgentState) -> AgentState:
    """replace_places Tool 결과를 받아서 상태 업데이트"""

    # 1. ToolResult 파싱
    data = get_last_tool_message(state["messages"]).content["data"]
    replaced_ids = data["replaced_place_ids"]
    new_places = data["places"]

    # 2. 기존 장소에서 제거
    current = state["last_recommended_places"]
    updated = _drop_places_by_ids(current, replaced_ids)

    # 3. 새 장소 추가
    updated.extend(to_simple_places(new_places))

    return {"last_recommended_places": updated}
```


>[!tip] 
이 계층이 등장하면서 기존의 `update_state_node` 하나에 모든 Tool 분기 로직이 모이는” 구조를 완전히 제거할 수 있었다.


### 4.4.  Router - 후처리 노드를 결정하는 분기 
> Tool 실행이 끝난 후 어떤 Node를 호출할지 결정하는 함수를 설정하는 것 <br>
> `route_after_tools()`가 Tool별로 적절한 후처리 노드로 라우팅

#### 4.4.1.  기존 문제점 : Before 💢
```python
# update_state_node.py - 모든 Tool 처리
def update_state_node(state):
    if tool_name == "replace_places":
        # 특수 로직 1
    elif tool_name == "create_travel_route":
        # 특수 로직 2
    else:
        # 일반 처리
```

- Tool이 늘어날 때마다 `if`문 추가
- 단일 파일에 모든 로직 집중
- Tool이 ToolResult 반환 → 순수하지 않음


#### 4.4.2.  개선 : After ✅
```python
# Tool -> 후처리 노드 매핑 (중앙 집중 관리)
TOOL_POSTPROCESSING_ROUTES: dict[Hashable, str] = {
    "replace_places": "handle_replace_places",
    "recommend_nearby_places": "handle_place_recommendation",
    "recommend_popular_places_in_region": "handle_place_recommendation",
    "recommend_places_by_all_users": "handle_workspace_recommendation",
    "create_travel_route": "handle_travel_route",
}
```
- Tool 이름 ➡ Node 이름으로 매핑 
- 기존에 상태 업데이트를 처리하던 node인 `Update_state_node`에 분기 되어있던 로직들이 분리되어 유지보수 용이

Tool 실행 후 `route_after_tools()`
```python
# Graph만들 때, workflow.add_conditional_edges("tools", route_after_tools, postprocess_edges)에서 사용한다 
def route_after_tools(state: AgentState) -> str:
    """
    Tool 실행 후 어떤 후처리 노드로 보낼지 결정
    각 Tool은 전용 후처리 노드가 상태 변경을 담당
    예시
    - replace_places → handle_replace_places
    - recommend_* → handle_place_recommendation
    - create_travel_route → handle_travel_route
    """
    last_tool_message = get_last_tool_message(state.get("messages", []))
    if not last_tool_message:
        return "agent"

    tool_name = getattr(last_tool_message, "name", "")
    target_node = TOOL_POSTPROCESSING_ROUTES.get(tool_name)
    if not target_node:
        return "agent"

    return target_node
```

### 4.5.  정리 

1. Service는 순수 비즈니스 로직 
2. Tool은 LLM 어댑터 
3. Node는 상태 관리 (Tool별 분리)
4. 명시적 라우팅 (route_after_tools)

## 5.  LangGraph 에이전트 만드는 예시 

`create_agent_graph` - LangGraph 에이전트 만드는 함수 
```PYTHON
# =========================
# 그래프 구성 (후처리 노드 분리 패턴)
# =========================
def create_agent_graph():
    """
    LangGraph 생성 - 후처리 노드 분리 패턴
    구조:
    1. Router → Agent → Tools (도구 실행)
    2. Tools → route_after_tools (라우터)
    3. route_after_tools → 각 Tool별 전용 후처리 노드
       - replace_places → handle_replace_places
       - recommend_* → handle_place_recommendation
       - create_travel_route → handle_travel_route
       - 기타 → agent (바로 복귀)
    4. 후처리 노드 → agent (상태 업데이트 후 복귀)
    """
		workflow = StateGraph(AgentState)
		
		# 노드 추가 
		# 엣지 추가 
    
    # tools -> route_after_tools (Tool별 후처리 노드로 분기)
    postprocess_edges: dict[Hashable, str] = {
        node: node for node in TOOL_POSTPROCESSING_ROUTES.values()
    }
    postprocess_edges["agent"] = "agent"  # 기본 경로
    workflow.add_conditional_edges("tools", route_after_tools, postprocess_edges)
    
    # todo : 하드코딩 부분 수정 
		# 각 후처리 노드 -> agent (상태 업데이트 후 에이전트로 복귀)
    workflow.add_edge("handle_replace_places", "agent")
    workflow.add_edge("handle_place_recommendation", "agent")
    workflow.add_edge("handle_workspace_recommendation", "agent")
    workflow.add_edge("handle_travel_route", "agent")
    
    memory = MemorySaver()
    return workflow.compile(checkpointer=memory)    
```


## 6.  이 패턴의 장점 정리 

### 6.1.  책임 분리 (Separation of Concerns)
- **Service**: "데이터를 어떻게 가져올까?"
- **Tool**: "LLM에게 데이터를 어떻게 전달할까?"
- **Node**: "가져온 데이터로 상태를 어떻게 바꿀까?"

### 6.2.  명시성 (Explicitness)
- Tool별 분기가 `route_after_tools()`에 명시적으로 표현
- `if tool_name == ...` 분기문이 여러 곳에 흩어지지 않음

## 7.  예시: replace_places 플로우

```
1. LLM이 replace_places Tool 호출 결정
   ↓
2. Tool Layer (@tool replace_places)
   - ReplacePlaceRequest DTO 생성 (6개 파라미터 캡슐화)
   - Service.find_replacement_places(request) 호출
   - ToolResult로 포장 (replaced_place_ids 메타정보 추가)
   ↓
3. route_after_tools() 라우터
   - tool_name == "replace_places" 확인
   - "handle_replace_places" 노드로 라우팅
   ↓
4. Node Layer (handle_replace_places_node)
   - ToolResult 파싱
   - 기존 장소에서 교체 대상 제거
   - 새 장소 추가
   - AgentState 업데이트
   ↓
5. Agent로 복귀
```


## 8.  번외 : 아직 남아있는 리팩토링 대상들 

아직도 코드가 몇개 맘에 안든다. 
특히 graph생성에서 노드명, 엣지명 등을 입력하는 과정에서 나는 여전히 하드코딩되어 있다. 고치고 싶은데.. 이제 더 이상 코드를 고치고 PR올리면 팀원들도 불안해 할 것 같기도 하고(다 끝난 마당에 왜 코드를 또 고치냐...이런...) 나도 지금 이력서 쓰고, 이전에 했던거 정리하고 면접, 코테 준비 등으로 바쁘다보니 이걸로 만족해야 할 것 같다.
![Pasted image 20251204153622.png](/img/user/supporter/image/Pasted%20image%2020251204153622.png)
AI Agent 아키텍처 이게 맞는지 모르겠다.
근데 이전에 주먹구구식으로 LangGraph관련 코드들을 작성하기 시작하면서 점점 코드를 개선해 나가는 과정이 좋은 경험이였던 것 같다. 


## 9.  번외2 - 어뎁터 패턴 쓰면서 느낀 한계와 부작용


- 이번 아키텍처에서 Tool Layer를 **“LLM 어댑터”**로 두고 3계층을 강하게 분리한 건 분명 장점이 많았다. 
- 하지만 막상 프로젝트 전체에 적용해보니, “무조건 좋은 패턴”이라고 말하기 어렵다는 생각이 들었다.

### 9.1.  복잡하고 번거로움 
작은 기능하나를 추가해도 Service - Tool - Node -Router 까지 다 건드려야 했다.
레이어가 너무 많다. 이러한 복잡성은 Agent Graph를 그리는 것에도 영향을 준다 
특히나 이번 프로젝트처럼 긴박한 조건이라면 더욱 더 불필요한 패턴이었던 것 같다.
동선이 너무 길어서 디버깅도 귀찮고....

### 9.2.  ToolResult 스키마로 인한 강한 결합

한 번 `ToolResult` 스키마를 정의해두니까 중간에 필드를 바꾸거나 구조를 바꾸기 어려워졌다. 중간에 ToolResult스키마 변경할만한게 있나? 생각할 때 변경을 생각했는데 Node에서 `data["..."]`를 파싱하는 코드가 여러 군데 존재하여 변경도 어렵게 느꼈다. <br>
결국, 표준화를 하다보니 동시에 “강한 결합”이 동반하게 되어 썩... 맘에 드는 코드가 된 것 같지는 않다.



