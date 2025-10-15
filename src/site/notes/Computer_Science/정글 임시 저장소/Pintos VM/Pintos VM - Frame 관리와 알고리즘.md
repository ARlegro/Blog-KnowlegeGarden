---
{"dg-publish":true,"permalink":"/Computer_Science/정글 임시 저장소/Pintos VM/Pintos VM - Frame 관리와 알고리즘/","noteIcon":"","created":"2025-10-08T16:00:48.804+09:00","updated":"2025-10-15T16:22:46.507+09:00"}
---



### 0.1.  목차

- [[#Frame 개념 및 관리|Frame 개념 및 관리]]
	- [[#Frame 개념 및 관리#1.  Frame 이란|1.  Frame 이란]]
	- [[#Frame 개념 및 관리#2.  Frame Table 이 필요한 이유|2.  Frame Table 이 필요한 이유]]
	- [[#Frame 개념 및 관리#3.  동기화 주의 - Frame Table 관리 시 주의|3.  동기화 주의 - Frame Table 관리 시 주의]]
- [[#페이지 교체 정책(Frame 관리) 알고리즘|페이지 교체 정책(Frame 관리) 알고리즘]]
	- [[#페이지 교체 정책(Frame 관리) 알고리즘#1.  LRU 알고리즘|1.  LRU 알고리즘]]
		- [[#1.  LRU 알고리즘#1.1.  개념 및 동작 원리|1.1.  개념 및 동작 원리]]
		- [[#1.  LRU 알고리즘#1.2.  장단점|1.2.  장단점]]
	- [[#페이지 교체 정책(Frame 관리) 알고리즘#2.  Clock 알고리즘  ⭐|2.  Clock 알고리즘  ⭐]]
		- [[#2.  Clock 알고리즘  ⭐#2.1.  개념|2.1.  개념]]
		- [[#2.  Clock 알고리즘  ⭐#2.2.  작동 방식|2.2.  작동 방식]]
		- [[#2.  Clock 알고리즘  ⭐#2.3.  장단점|2.3.  장단점]]
	- [[#페이지 교체 정책(Frame 관리) 알고리즘#3.  Second Chance|3.  Second Chance]]
- [[#Frame 관리 구현|Frame 관리 구현]]
	- [[#Frame 관리 구현#1.  Frame Table init|1.  Frame Table init]]
		- [[#1.  Frame Table init#1.1.  자료 구조 선택 고민 🤔|1.1.  자료 구조 선택 고민 🤔]]
		- [[#1.  Frame Table init#1.2.  구현|1.2.  구현]]
	- [[#Frame 관리 구현#2.  활용 - vm_get_victim|2.  활용 - vm_get_victim]]
		- [[#2.  활용 - vm_get_victim#2.1.  코드 및 동작 흐름|2.1.  코드 및 동작 흐름]]
		- [[#2.  활용 - vm_get_victim#2.2.  번외 : 왜 owner_pml4를 활용할까?|2.2.  번외 : 왜 owner_pml4를 활용할까?]]
	- [[#Frame 관리 구현#3.  활용 - vm_get_frame()|3.  활용 - vm_get_frame()]]

---
# Frame 개념 및 관리 

VM 최종 Frame 구조체 
```c
// 메타데이터 구조체 (관리용)
struct frame {
  void *kva;
  struct page *page;
  struct list_elem elem;
  uint64_t *owner_pml4;
};
```


---
## 1.  Frame 이란 
> 물리 메모리의 페이지 단위 조각 
- RAM을 페이지 단위로 나눈 블록
- 각 Frame은 커널 가상 주소로 1:1 매핑된다 ➡커널 직접 접근 가능 
- 프로세스의 가상 Page가 실제 물리 공간과 연결될 때 Frame이 실체화된다(Frame은 그냥 메타데이터 느낌)

---
## 2.  Frame Table 이 필요한 이유 

> [!info] Frame Table: 모든 물리적 Frame들의 상태를 저장한 자료구조
> OS의 메모리 관리 핵심 모듈로 대표적으로는 아래 기능을 한다.
> 1. *추적 기능*
> 	- 각 Frame이 어떤 페이지에게 or 어떤 Thread에서 사용되는지 추적 가능 
> 	- OS는 **모든 Frame들을 추적하고 관리하기 위해** Frame Table을 둔다 
> 2. *매핑 유지*
> 	- **가상 메모리 → 실제 메모리** 매핑을 유지시켜준다.


아래와 같은 효과를 얻을 수 있음
1. *물리 메모리 할당과 해제*
	- 새로운 페이지 Fault 발생 ➡ Frame Table에서 빈 frame 찾고 할당 가능 
	- 메모리 부족 시 ➡ 교체 알고리즘으로 Frame 반환도 가능 
	  
2. *Eviction(교체) 대상 선택* 
	- **각 Frame의 접근 기록을 기준으로 어떤 Frame을 쫓아낼지 결정 가능**하다(최근 접근 여부, pinned 상태 등)
	- 만약 Frame Table이 없으면 어떤 Frame이 최근 접근되었는지 추적이 불가능 ➡Eviction 정책 어케할지 미흡 
	  	  
3. *역방향 매핑 추적* 
	- 해당 Frame을 어떤 page or thread가 갖고 있는지 추적하기 쉽다
	- 이는 프로세스 종료 시 관련 Frame 회수 시 용이하다.

>[!danger] Frame 테이블 관리 안하면?
>Frame Table을 관리하지 않으면 OS 전체의 메모리 일관성이 깨진다.
>1. **교체(eviction) 정책 카오스**
>	- 어떤 Frame이 최근에 쓰였는지 등에 대한 정보를 몰라서 교체 정책을 어케 써야될지 모호하다. 
>2. **Memory Leak** : 프로세스 종료 시 어떤 Frame을 회수해야될지 모호함 
>3.**동시성 문제**
>	- 여러 Page가 동시에 Fault 발생 시 Frame 중복 할당이 가능하다.

---
## 3.  동기화 주의 - Frame Table 관리 시 주의
- 여러 스레드가 동시에 페이지 fault를 일으킬 수 있으니, frame table 접근 시 `frame_lock` 같은 전역 락으로 동기화 必
- 중복 할당, race condition 방지.


---
# 페이지 교체 정책(Frame 관리) 알고리즘 

- Frame 테이블에서 사용되지 않고 있는 Frame을 획득하는 것이 중요 
- 근데 모두 사용되고 있다면? 몇몇 페이지들을 frame에서 쫓아내어 그 frame을 free상태로 만들어주는 것이 필요하다.
- **문제는 그 Frame을 뭘로 결정하냐**는 것이다.
- 이 결정 알고리즘의 대표적인 3가지만 볼 것
	- LRU
	- Clock
	- Second-Chance

---
## 1.  LRU 알고리즘  
#지역성 
LRU = Least Recently Used

---
### 1.1.  개념 및 동작 원리
> 가장 오래 접근되지 않은 페이지를 희생양으로 고르는 정책 

- *가정* : 최근에 자주 접근된(Accessed bit) 페이지는 또 접근될 가능성이 높고, 오래 안 쓰인 페이지는 앞으로도 쓸 확률이 낮다 

- *동작 원리* 
	1. **각 페이지가 마지막으로 접근된 시간/순서를 저장**
		- 각 Frame들이 "언제 마지막으로 접근되었는지"를 추적해야 함 
		- Accessed Bit나 Timestamp를 활용할 수 있다.
		  
	2. **교체 시점에 가장 오래된 페이지를 제거** 

---
### 1.2.  장단점 

*✅장점 - 가장 정확*
- *이론적으로 교체 품질이 우수* : 매 접근마다의 최근 사용 시점을 갱신하고 그것을 활용하기에 

*💢단점 *
- *재정렬 비용* : 매 접근마다 오래된 순서로 정렬하는 비용 문제 (매 접근마다 O(n))
	- Frame의 개수가 커질수록 오버헤드가 심해질 수 밖에 없다.
- *타임스탬프 갱신 귀찮* : 특히 pintos에서는 더욱
- 다중 주소 공간에서 *접근 시각 동기화 어려움*

재정렬 비용이랑 타임스탬프 갱신 때문에 별로인듯?

---
## 2.  Clock 알고리즘  ⭐
페이지 교체 정책 알고리즘 중 하나로, **물리 메모리가 가득 찼을 때 다음에 사용될 가능성이 가장 낮은 Frame을 선정하여 페이지 교체**

Pintos에서 나는 이 알고리즘으로 Frame 테이블을 관리했다.

---
### 2.1.  개념 
> 1. Frame들을 원형 리스트로 관리
> 2. 시계 바늘처럼 순회하며 최근에 사용되지 않은 Frame을 찾는 방식 by Accessed Bit

---
### 2.2.  작동 방식 
[출처 : Clock Page Replacement 유튜브](https://www.youtube.com/watch?v=b-dRK8B8dQk)
![Pasted image 20251013112349.png](/img/user/supporter/image/Pasted%20image%2020251013112349.png)

1. 특정 Frame을 가리키는 포인터(시계 바늘같은)가 있다.
2. 해당 `Frame`의 `Accessed bit`를 확인
	- 1이면 : 최근에 접근된 것 ➡ 0으로 reset하고 다음 Frame으로 이동
	- 0이면 : Eviction 대상 
	- 결국 한바퀴 돌면 누군가는 무조건 0이므로 무조건 Eviction 대상 고를 수 있다.
	  
3. Eviction 수행 및 바늘을 다음 Frame으로 옮기기 

---
### 2.3.  장단점 

1. *✅장점*
	- **구현의 단순성** : 원형 리스트와 포인터 하나만으로 구현되어 매우 간단
	- **낮은 오버헤드** : 페이지 접근 시간 기록을 매번 갱신할 필요가 없고 단순 PTE비트만 확인하고 수정하면 되므로 오버헤드가 적다
	  
2. *💢단점*
	- 부정확할 수 있다 Cuz 완전한 LRU가 아님 


---
## 3.  Second Chance


> 'FIFO + accessed bit 확인'으로 한 번의 기회를 주는 방식 

>[!QUESTION] accessed bit 정체
> - CPU의 H/W PTE에 있는 bit 중 하나 (Pintos에서는 Page 구조체에 담으면 될 듯?)
> - CPU가 이 가상 Page를 읽거나 썼을 떄 H/W가 자동으로 1로 세팅 
> 


*동작 원리*
1. 큐(또는 리스트)의 **맨 앞**에서 희생 후보를 고른다.  
2. 후보의 **Accessed(A) bit** 확인:
    - A=1 → **두 번째 기회** 부여: A를 0으로 클리어하고 **맨 뒤로** 보낸다.
    - A=0 → **victim 확정**, 교체.
        
3. 반복

*✅장점* 
- 구현이 매우 쉬움 (품질도 그렇게 나쁘지 않음)
- 접근 비트만 사용해서 구현 가능 

*💢단점*
- 최악의 경우 스캔 비용 多 : Accessed bit가 1인 페이지가 많으면 여러 번 순환해야 함 
- LRU 보다는 정확도가 떨어질 수 있다.


---
# Frame 관리 구현 

## 1.  Frame Table init
---
### 1.1.  자료 구조 선택 고민 🤔
> 어떤 자료구조를 선택할 것인가❓

`init`은 기존 스켈레톤 코드들을 생각하면 크게 어려운 점은 아니다.
문제는 어떤 자료구조를 사용할 것이냐는 점이다.
Pintos에서 제공하는 스켈레콘 자료구조는 
1. List
2. Hash
3. Bitmap

정도?였던 것 같다.

각 자료구조를 생각했을 때 Frame Table은 List로 구현하는게 맞는 것 같다.<br>
*List가 유리한 이유*
1. *삽입 삭제가 잦음*
	- 프로세스의 생성/종료 시 뿐만 아니라 페이지 로드/Evict 시마다 **`frame`의 추가/제거가 잦다.**
	- Pintos에서 구현된 List는 **연결리스트 구조이므로 삽입/삭제 시 유리할 것**이다.
	  
2. *Frame 정보 구조체를 활용하기에 좋음*
	- Frame에 대한 정보를 보관하기 위해서는 0, 1만 표현하는 `Bitmap` 구조는 별로이다.
	- 반면 List는 elem이라는 필드를 활용해서 구조체의 정보를 담을 수 있다.
	```c
	/* The representation of "frame" */
	// 메타데이터 구조체 (관리용)
	struct frame {
	  void *kva;
	  struct page *page;
	  struct list_elem elem;
	  uint64_t *owner_pml4;
	};
	```
	  
3. *Eviction 알고리즘과 자연스럽다*
	- Swap In/Out에서는 `Eviction`로직을 통해 특정 Frame을 메모리에서 쫓아내야 한다.
	- 이 때, 특정 Frame을 고르는 알고리즘들(LRU, Clock 등)을 선택하는데 이는 순차 탐색을 전제로한다.
	- 따라서 이게 유리
	  
4. *단순*
	- Hash로 구현할 수는 있지만 충돌 관리나 디버깅 시 List가 용이하다고 봤다.
	  
---
### 1.2.  구현 

> vm_init() 에서 구현 - 컴퓨터 부팅 시

어디서 init()을 시킬지 고민해보면 아주 간단하다.
`vm.c` 맨 위에는 컴퓨터 부팅 시 `vm`관련 함수들을 초기화하는 함수가 있다. 여기에 추가하면 될 것이다. 또한 clock알고리즘을 사용할 것이기 때문에 관련 포인터를 초기화하는 로직이 필요할 것이다.
```c
struct list frame_table;
struct lock frame_lock;
struct list_elem* clock_ptr;  

/* Initializes the virtual memory subsystem by invoking each subsystem's
 * intialize codes. */
void vm_init(void) {
  vm_anon_init();
  vm_file_init();
  register_inspect_intr();
  /* DO NOT MODIFY UPPER LINES. */
  list_init(&frame_table); // table 시작 
  lock_init(&frame_lock); // lock 시작 
  clock_ptr = list_begin(&frame_table);  // clock 알고리즘 용 pointer 초기화
}
```

---
## 2.  활용 - vm_get_victim
> Clock 알고리즘 기반으로 희생자 Frame을 선택하는 함수

---
### 2.1.  코드 및 동작 흐름
이 함수는 물리 메모리(Frame Table)가 가득 찼을 때, 교체(Eviction) 대상이 될 Frame을 선택하는 함수이다. 이 때, Clock알고리즘을 기반으로 하며 CPU에서 추적해줬던 access bit를 순차적을 확인할 것 
```C
static struct frame* vm_get_victim(void) {
  struct frame* victim = NULL;
  struct list_elem* e = NULL;
  uint64_t* pml4 = thread_current()->pml4;
  /* TODO: The policy for eviction is up to you. */
  // 애초에 Frame이 없는 경우 
  if (list_empty(&frame_table)) {
    return NULL;
  }

  // 1. 락 획득 - Frame 돌기 위해 락 걸기 Cuz 동시성 문제 방지 
  lock_acquire(&frame_lock);
  // 원형 순회 Until 희생자가 나올 때까지 
  while (victim == NULL) {
    struct frame* candidate = NULL;
    
    // 2. Clock 포인터를 기준으로 순회 시작
    for (e = clock_ptr; e != list_end(&frame_table); e = list_next(e)) {
      candidate = list_entry(e, struct frame, elem);
      if (candidate == NULL) continue;
      // 각 Frame의 접근 비트 확인
      if (pml4_is_accessed(candidate->owner_pml4, candidate->page->va)) {
        pml4_set_accessed(candidate->owner_pml4, candidate->page->va, false);
      } else {
        victim = candidate;
        break;
      }
    }

		// 4. 다음 순회 지점 갱신
    clock_ptr = victim ? list_next(e) : list_begin(&frame_table);
  }
  lock_release(&frame_lock);
  return victim;
}
```
*동작 흐름*
1. *락 획득*
	- Frame Table은 모든 스레드가 공유하므로,  
	- 동시 접근으로 인한 race condition을 방지하기 위해 `frame_lock`을 사용한다.
	  
2. *Clock 포인터를 기준으로 순회 시작*
	- `clock_ptr`는 원형 리스트에서 “시계 바늘”처럼 동작한다.
	- 리스트의 끝까지 탐색한 뒤에도 희생자를 못 찾으면, 다시 처음부터 탐색한다.
	  
3. *각 Frame의 접근 비트 확인*
	- 접근 비트가 **1(최근에 접근됨)** → 0으로 초기화하고 다음 프레임으로
	- 접근 비트가 **0(최근에 접근되지 않음)** → 해당 프레임을 **희생자(victim)** 로 선택
	  
4. *다음 순회 지점 갱신*
	```c
	clock_ptr = victim ? list_next(e) : list_begin(&frame_table);
	```

---
### 2.2.  번외 : 왜 owner_pml4를 활용할까? 

>[!QUESTION] 왜 owner_pml4를 활용할까? 
>그냥 `uint64_t* pml4 = thread_current()->pml4;`를 사용하지 않고 **왜 따로 Frame구조체에 `owner_pml4`를 저장하고 활용할까❓**
>1. Frame은 thread_current 의 것이 아닐 수 있다.
>	- 그 Frame을 `pml4`에 매핑시킨 `thread`랑 `vm_get_victim`을 호출시킨 `thread`랑은 다르다
>	- `thread_current()->pml4`는 “지금 실행 중인 프로세스의 페이지 테이블”에 대한 포인터일 뿐, **victim 후보의 실제 페이지 테이블이 아니다.**
>	  
>2. `Access Bit`은 **해당 프로세스의 pml4에서만 확인 가능**하다.
>	- `vm_get_victim`을 호출하는 `thread`와 `victim`을 고르기 위해 사용하는 `pml4`관련 메서드(`pml4_is_accessed` or `pml4_setaccessed`)에 넘겨야 되는 pml4가 다르기 때문이다.
>	- 만약 `thread_current()->pml4`를 사용하면, 현재 스레드의 페이지 테이블 기준으로 lookup 하므로  **다른 프로세스(쓰레드)가 소유한 페이지의 접근 여부를 전혀 알 수 없다.**

---
## 3.  활용 - vm_get_frame()
> 새로운 Frame을 요청받았을 때, 해당 Frame을 생성하고 Frame Table에 등록하는 함수

```c
struct frame* vm_get_frame(void) {
  struct frame* frame = NULL;
  void* kva = palloc_get_page(PAL_USER | PAL_ZERO);
  if (kva == NULL) {
    // 만약 더 이상 할당 가능한 물리 메모리가 없다면, vm_evict_frame()을 호출해 기존 프레임을 흐생(eviction) 한 뒤 재사용
    frame = vm_evict_frame();
  } else {
    frame = malloc(sizeof(struct frame));
    if (frame == NULL) {
      palloc_free_page(kva);
    } else {
      frame->kva = kva;
      frame->page = NULL;
    }
  }

  ASSERT(frame != NULL);
  ASSERT(frame->page == NULL);
  frame->owner_pml4 = thread_current()->pml4;
  // ✅ frame 테이블에 push
  list_push_back(&frame_table, &frame->elem);
  return frame;
}
```
특정 페이지에 매핑할 `frame`을 요청하면 `vm_get_frame()`이 호출된다.
이 때, 할당한 `frame`을 `frame_table`에 `push`하는 작업이 必


