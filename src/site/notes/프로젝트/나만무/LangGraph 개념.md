---
{"dg-publish":true,"permalink":"/프로젝트/나만무/LangGraph 개념/","noteIcon":"","created":"2025-12-03T14:53:06.940+09:00","updated":"2025-12-09T17:20:11.107+09:00"}
---


*LangGraph 관련* 
- [[프로젝트/나만무/LangGraph 개념\|LangGraph 개념]]
- [[프로젝트/나만무/LangGrpah 실전 패턴\|LangGrpah 실전 패턴]]
- [[프로젝트/나만무/LangGraph vs LangChain\|LangGraph vs LangChain]]
- [[프로젝트/나만무/LangGraph 핵심 기술 (state + checkpointer)\|LangGraph 핵심 기술 (state + checkpointer)]]
- [[프로젝트/나만무/LangGraph - Checkpointer 도입\|LangGraph - Checkpointer 도입]]
- [[프로젝트/나만무/LangGraph 에러 및 어려움\|LangGraph 에러 및 어려움]]



### 0.1.  목차
- [[#1.  왜 LangGrap인가?(공부/도입 이유)|1.  왜 LangGrap인가?(공부/도입 이유)]]
	- [[#1.  왜 LangGrap인가?(공부/도입 이유)#1.1.  LLM 서비스가 복잡해지면서...|1.1.  LLM 서비스가 복잡해지면서...]]
	- [[#1.  왜 LangGrap인가?(공부/도입 이유)#1.2.  단순 LangChain의 한계|1.2.  단순 LangChain의 한계]]
- [[#2.  LangGraph 개념 - Workflow Engine|2.  LangGraph 개념 - Workflow Engine]]
	- [[#2.  LangGraph 개념 - Workflow Engine#2.1.  기본|2.1.  기본]]
	- [[#2.  LangGraph 개념 - Workflow Engine#2.2.  핵심 개념|2.2.  핵심 개념]]
	- [[#2.  LangGraph 개념 - Workflow Engine#2.3.  Node, Edge 설계(간단히 - 코드 없이)|2.3.  Node, Edge 설계(간단히 - 코드 없이)]]
		- [[#2.3.  Node, Edge 설계(간단히 - 코드 없이)#2.3.1.  Node 설계 (각 단계의 모듈)|2.3.1.  Node 설계 (각 단계의 모듈)]]
		- [[#2.3.  Node, Edge 설계(간단히 - 코드 없이)#2.3.2.  Edge 설계 (흐름 제어)|2.3.2.  Edge 설계 (흐름 제어)]]
- [[#3.  LangGraph가 해결하는 주요 문제들 (장점)|3.  LangGraph가 해결하는 주요 문제들 (장점)]]
	- [[#3.  LangGraph가 해결하는 주요 문제들 (장점)#3.1.  장기 대화의 기억 문제 해결|3.1.  장기 대화의 기억 문제 해결]]
	- [[#3.  LangGraph가 해결하는 주요 문제들 (장점)#3.2.  Multi-step|3.2.  Multi-step]]
- [[#4.  LangGraph핵심 기술 (state + checkpointer)|4.  LangGraph핵심 기술 (state + checkpointer)]]
- [[#5.  LangGraph 실전|5.  LangGraph 실전]]
- [[#6.  번외 - 구체적인 LangChain vs LangGraph|6.  번외 - 구체적인 LangChain vs LangGraph]]


[LangGrpah 공식 사이트](https://docs.langchain.com/oss/python/langgraph/overview?_gl=1*3behwk*_gcl_au*Nzc3NjIyODAuMTc2MzcyMDc3Mw..*_ga*NTY1NDQ2MDQxLjE3NjM3MjA3NzQ.*_ga_47WX3HKKY2*czE3NjM3MjA3NzQkbzEkZzAkdDE3NjM3MjA3NzQkajYwJGwwJGgw)

## 1.  왜 LangGrap인가?(공부/도입 이유)
### 1.1.  LLM 서비스가 복잡해지면서...
LLM 기반 서비스가 고도화 될수록 아래 요구사항들이 생긴다.
- 이전 **대화/상태를 오래 기억하고 재활용**해야 한다.
- 여러 **도구(tool)를 여러 번 호출하면서 서로 협업**시켜야 한다.
- 유저 입력이나 중간 결과에 따라 **흐름(분기/루프)을 유연하게 바꿔야**한다.

이러한 요구사항이 쌓이면 단순한 LLM 호출 ➡ 도구 호출 ➡ 답변 수준을 넘어선 **"복잡한 워크플로우(상태 + 분기 + 관리)를 관리해야 한다"**


### 1.2.  단순 LangChain의 한계 

초기 단순 LangChain만으로 복잡한 에이전트를 만들 때 아래의 문제를 많이 겪었다💢
1. **도구 호출 스텝이 꼬이기 쉽다**
   - Flow를 “함수 호출 흐름”으로만 표현하다 보니 `if/else`, 반복, 예외 처리 로직이 섞이면서 코드가 금방 스파게티가 된다.
   - 여러 번 도구를 호출하고, 중간 결과에 따라 다시 LLM을 부르는 패턴을 구현하기가 점점 어려워진다.

2. **상태(state)를 구조적으로 다루기 어렵다**
   - LangChain의 기본 메모리는 대부분 “메시지 히스토리 append” 수준이다.
   - `last_recommended_places`, `excluded_place_ids`, 유저 설정 같은 **구조화된 상태**를  
     여러 스텝에 걸쳐 안전하게 관리하려면 개발자가 직접 DB/캐시 + 머지 로직을 만들어야 한다.(개발 복잡성)

3. **장기 대화에서 일관성이 쉽게 깨진다**
   - 히스토리 양이 늘어나면, 어느 시점의 어떤 정보가 중요한지 관리하기가 어렵다.
   - “어떤 도구 결과를 기준으로 지금 이 답변을 하는지”를 추적하기 힘들다.

4. **실패 복구/재시도 로직이 체계적으로 없다**
   - 중간 도구가 실패했을 때 “조금 전 상태로 되돌려서 다시 시도하는” 구조를 만들려면  
     별도의 상태 스냅샷/세이브 로직을 직접 구현해야 한다.

>[!TIP] 이런 복잡성을 **상태 머신 + 그래프 형태로 추상화해서 해결**하는 프레임워크가 LangGraph다.

---

## 2.  LangGraph 개념 - Workflow Engine
❌단순 LangChain 확장판이 아니다❌
✅LangChain 생태계를 활용하지만, **"상태머신 + 그래프 기반 workflow engine"**✅

### 2.1.  기본
> 여러 단계의 LLM/도구 호출을 **Graph(상태 머신)** 로 정의하고,  
> 그 그래프 전반의 상태를 일관되게 관리해주는 프레임워크

- 철학 : 에이전트의 workflow를 그래프(Node/Edge)로 정의 
- 각 단계는 Node(함수)로 구성되면 Node간 연결은 Edge로 설정한다.
- **그래프 전체가 하나의 State(상태)를 공유하는 상태 머신**이다.
- 보통 LangChain의 LLM/Tool/Message 타입을 그대로 재사용하지만, 반드시 LangChain에만 묶이진 않는다.

![Pasted image 20251121192823.png](/img/user/supporter/image/Pasted%20image%2020251121192823.png)


### 2.2.  핵심 개념 

1. *그래프 구조* 
	- 복잡한 비선형 흐름을 명시적으로 표현 (Node, Edge)
	- 조건 분기, 루프, 반복 실행을 코드가 아닌 "그래프 형태"로 설계할 수 있다.
	  
2. *상태 관리*
	- 모든 Node는 공통 State(딕셔너리)를 읽고/쓰고/업데이트한다.
	- 각 필드는 “어떻게 업데이트할지”를 정하는 **Reducer**를 가질 수 있다. (ex. `messages`는 쌓이도록(add), `intent`는 마지막 값으로 덮어쓰도록)
	- 이 덕분에 **장기 대화, 여러 단계 의사결정, 도구 재시도 같은 복잡한 multi-step에 잘 맞는다**

### 2.3.  Node, Edge 설계(간단히 - 코드 없이)
이번에 크래프톤 정글 나만무 프로젝트에서 설계한 node, edge를 기준으로 아래와 같이 모듈화하고 흐름을 제어했다.

#### 2.3.1.  Node 설계 (각 단계의 모듈)
> 모든 단계를 Node(모듈)로 쪼개는 과정 

Node는 SRP원칙마냥 딱 한가지 역할만 담당하는 것이 좋다 한다.

TODO : 그림 

- `router_node`: 의도 분류
- `agent_node`: LLM 판단 ➡ LangGraph의 핵심 두뇌
- `tool_node`: 실제 도구 실행
- `update_state_node`: 결과를 상태에 반영하는 노드 
- `refine_exclude_node`: 특정 작업(제외 작업) 처리하는 노드 

→ 모든 기능을 Node 단위로 분리  
→ 유지 보수성 극대화

*간단 예시 - 사용자 의도파악 및 연결*
- 입력 Node로 입력이 들어오면

#### 2.3.2.  Edge 설계 (흐름 제어)
Node사이의 흐름을 제어하는 역할
**Node간 이동은 의도 분류 or 상태 값을 기반으로 Edge가 제어**한다.
```python
# 예시 
   # router -> agent or refine_exclude (의도에 따라 분기)
    workflow.add_conditional_edges(
        "router",
        route_by_intent,
        {"agent": "agent", "refine_exclude": "refine_exclude"},
    )

    # agent -> tools or END (도구 호출 여부에 따라 분기)
    workflow.add_conditional_edges(
        "agent", should_continue, {"tools": "tools", END: END}
    )

    # tools -> update_state (도구 실행 후 상태 업데이트)
    workflow.add_edge("tools", "update_state")
```
- router → agent
- agent → tool or END
- tool → agent
- refine → agent


## 3.  LangGraph가 해결하는 주요 문제들 (장점)

> LangGraph는 **상태 관리 및 에이전트 조정 관련된 복잡성을 추상화**한다.


### 3.1.  장기 대화의 기억 문제 해결 

*❗LangChain에서는* 
- **대화 history가 단순 append수준**이다.
- `Langchain`을 쓰다 보면 이전 기억을 제대로 활용 하지 못하는 경우가 잦을 것이다.
	- 어떤 정보가 중요한지 구분 못함
	- 이전 단계의 출력/중간상태를 구조화해 저장하기 힘듬
	- 여러 스텝을 거친 후 일관된 reasoning 유지 힘듬 
- 물론 LangChain을 써서도 어찌저찌 구조화 할 수 있지만 이를 제대로 관리하고 활용하기 위해서는 개발자가 여러 작업을 해야한다.💢 이는, 오류 발생 가능성을 증가시키고 개발 생산성을 저하시킬 수 있다. 💢

*✅LangGraph에서는* 
- 구조화된 dictionary형태로 해결(State)
- `LangGraph`는 **상태 전체를 구조화하는 객체를 만들 수 있어 장기 대화 시 정확한 reasoning을 유지**할 수 있다.
- 메시지 외에도  
	- intent  
	- 추천 결과 목록  
	- 제외할 id  
	- 유저 설정  
	  같은 정보를 필드 단위로 구조화해서 유지할 수 있다.
- 각 Node는 이 State를 읽고/쓰고/업데이트하므로,  장기 대화에서도 **일관된 reasoning**이 가능하다.

### 3.2.  Multi-step
*❗LangCahin에서는*
- 에이전트가 **도구를 하나 실행하고 끝**나는 구조가 대부분이다. 💢
	- LLM ➡ 도구 한 번 실행 ➡ 종료 패턴 
- 여러 도구를 연속적으로 실행하고 그 결과를 바탕으로 판단 후 다른 도구를 호출하는 tool-loop 패턴을 만들기 힘들다.
	- 프롬프트를 강하게 걸어서 겨우겨우 불러오게 할 수 있을수도 있겠지만 내 경험상 이는 쉽지 않았다.

*✅LangGraph에서는*
- agent_node를 중심으로 반복적으로 reasoning을 하며 여러 노드 간의 협업을 명확하게 표현한다.
- 또한, **오류 발생 가능성을 매우 줄여준다**.(Cuz 합리적인 추론 ＋ 추론 기반 재시도)
	- Langgraph가 실수로 tool에 잘못된 인자를 전달하고 머리 역할을 하는 agent_node가 오류를 전달 받았을 때, **재 reasoning을 하며 올바른 인자로 다시 tool을 호출**하는 경우도 있었는데 이는 LangChain의 구조에서는 힘들었던 것

## 4.  LangGraph핵심 기술 (state + checkpointer)
[[프로젝트/나만무/LangGraph 핵심 기술 (state + checkpointer)\|LangGraph 핵심 기술 (state + checkpointer)]]

## 5.  LangGraph 실전

[[프로젝트/나만무/LangGrpah 실전 패턴\|LangGrpah 실전 패턴]]

## 6.  번외 - 구체적인 LangChain vs LangGraph
[[프로젝트/나만무/LangGraph vs LangChain\|LangGraph vs LangChain]]