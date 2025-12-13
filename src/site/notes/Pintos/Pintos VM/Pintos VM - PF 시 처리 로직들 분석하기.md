---
{"dg-publish":true,"permalink":"/Pintos/Pintos VM/Pintos VM - PF 시 처리 로직들 분석하기/","noteIcon":"","created":"2025-12-03T14:52:52.877+09:00","updated":"2025-12-13T09:26:33.313+09:00"}
---

PF = Page Fault

### 0.1.  목차

[[#0.2.  서론|0.2.  서론]]

[[#1.  vm_try_handle_fault|1.  vm_try_handle_fault]]
- [[#1.  vm_try_handle_fault#1.1.  초기 코드 및 추측|1.1.  초기 코드 및 추측]]
- [[#1.  vm_try_handle_fault#1.2.  관련 코드들|1.2.  관련 코드들]]
- [[#1.  vm_try_handle_fault#1.3.  세미 완성본|1.3.  세미 완성본]]
- [[#1.  vm_try_handle_fault#1.4.  추가 고민|1.4.  추가 고민]]
- [[#1.  vm_try_handle_fault#1.5.  번외 : 최종 코드|1.5.  번외 : 최종 코드]]

 [[#2.  초기 vm_do_claim_page 세팅|2.  초기 vm_do_claim_page 세팅]]
- [[#2.  초기 vm_do_claim_page 세팅#2.1.  사전 개념 : 유저 VA vs 커널 VA|2.1.  사전 개념 : 유저 VA vs 커널 VA]]
- [[#2.  초기 vm_do_claim_page 세팅#2.2.  관련 메서드 - pml4_set_page|2.2.  관련 메서드 - pml4_set_page]]
- [[#2.  초기 vm_do_claim_page 세팅#2.3.  관련 메서드 -  vm_get_frame()|2.3.  관련 메서드 -  vm_get_frame()]]
- [[#2.  초기 vm_do_claim_page 세팅#2.4.  Frame 테이블 관리|2.4.  Frame 테이블 관리]]
	- [[#2.4.  Frame 테이블 관리#2.4.1.  GitBook Frame Table 관리 글|2.4.1.  GitBook Frame Table 관리 글]]
	- [[#2.4.  Frame 테이블 관리#2.4.2.  초기 고민 🤔|2.4.2.  초기 고민 🤔]]
	- [[#2.4.  Frame 테이블 관리#2.4.3.  실제 관리 및 구현|2.4.3.  실제 관리 및 구현]]


### 0.2.  서론 

`VM`을 다루다보면 `Page`를 미리 `Load`하지 않기 때문에 `Page_Fault`가 나고 exception을 일으킨다.
`exception.c`파일의 `page_fault`함수 내부를 보면 아래와 같은 코드가 준비되어 있다.
```c
static void page_fault (struct intr_frame *f) {

#ifdef VM
  /* For project 3 and later. */
  if (vm_try_handle_fault (f, fault_addr, user, write, not_present))
    return;
#endif
```
VM으로 `Page_Fault`를 다루는 것이 중요한 부분이 될 것이다.
이와 관련된 미리 세팅된 코드들과 관련 코드들을 분석하고 어떻게 해결해 나갈지를 추론하는 시간을 가질 것이다.

> [!INFO] Preview
> 1. Page Fault 발생
> 2. `vm_try_handle_fault()` 함수에서 핸들 
> 	- 각종 검증 
> 	- SPT에서 해당 가상주소에 해당하는 page 있는지 검색 
> 	- Page가 있다면 stack_growth대상인지 확인하고 stack늘려주거나 말거나
> 	- Page가 없다면❓
> 		- `vm_do_claim_page()`로직 수행 : frame얻기, pml4 세팅, swap_in

---
## 1.  vm_try_handle_fault
---
### 1.1.  초기 코드 및 추측 
```c
bool vm_try_handle_fault (struct intr_frame *f UNUSED, void *addr UNUSED,
    bool user UNUSED, bool write UNUSED, bool not_present UNUSED) {
    
  struct supplemental_page_table *spt UNUSED = &thread_current ()->spt;
  struct page *page = NULL;

  /* TODO: Your code goes here */
  return vm_do_claim_page (page);
}
```
- `vm_try_handle_fault`함수 내부를 보면 `todo`라는 주석과 함께 위와 같은 코드들이 준비 되어 있다.
- *추측*
	- `supplemental_page_table_init` 에서 구현된 `spt`를 활용하는 것 같고 이를 기반으로 `vm_do_claime_page`를 호출하는 것 같다.
	- 즉, spt -> page 추출 이 필요하다고 간단히 예측할 수 있다.
	- 그렇기 때문에 이거 관련된 스켈레톤 코드들을 찾아야 할 것이다.
	  
- *뭘 처리해야 하는가❓*
	- 우선 유효 검증 : 유저에서 접근한 주소인지, 실존하는 주소인지 
	- 그다음은❓
		- 일단 `#PF`가 났다는 것은 메모리에 올라와 있지 않다는 것이다.
		- 이 함수가 어디까지의 책임이 있을까를 알려면 마지막에 선언된 `vm_do_claim_page`를 확인해 봐야 한다. 
		- 확인 결과 `vm_do_claim_page`은 **인자로 주어진 page에 물리 메모리 프레임을 할당하는 역할**을 하는 함수였다.
	- 즉, 이 함수 내에서는 메모리에 올릴 page만 세팅하면 될 것이다.

`spt`로부터 `va`에 해당하는 `page`를 찾는다.
- *있다면*❓- 그냥 뒤의 `vm_do_claim_page`함수에 넘겨주면 끝난다
- *없다면*❓- 스택 확장 必
	- 새 스택 page를 grow할 수 있는지 확인하고 여부에 따라 결정 


---
### 1.2.  관련 코드들
✔`spt_find_page()` - SPT에서 VA에 해당하는 페이지 찾는 함수 
```c
struct page *spt_find_page (struct supplemental_page_table *spt UNUSED, void *va UNUSED) {
  struct page *page = NULL;
  /* TODO: Fill this function. */
  return page;
}
```
- 위의 코드는 초기 세팅되어 있는 `spt_find_page`함수이다.
- *추측* 
	- 일단 직접 구현하기 전, spt관련 스켈레톤 코드가 있는지 생각해볼 수 있다. 
	- spt는 `hash_table`로 구현했기에 `hash.h`에 적힌 스켈레톤 코드도 참고하면 좋을 것 같다.
	- `hash_find`라는 정의된 함수가 있긴 한데, 이거는 인자로 `hash_elem`이 필요하네... 지금 함수에서 넘어온 것은 `VA`인데? `hash_elem`은 page 구조체에 있는 필드이다.
	- 그렇다면 일단 va를 기반으로 page를 찾는 함수를 찾아야겠다.(근데 못 찾았다)
	- 그러던 중 `hash_find()`의 **반환값과 인자의 이상한 점을 발견**⭐했고 이를 활용하고자 했다.

✔`hash_find()` 내부 동작
```c
/* Finds and returns an element equal to E in hash table H, or a
   null pointer if no equal element exists in the table. */
struct hash_elem *
hash_find (struct hash *h, struct hash_elem *e) {
  return find_elem (h, find_bucket (h, e), e);
}
  
/* Searches BUCKET in H for a hash element equal to E.  Returns
   it if found or a null pointer otherwise. */
static struct hash_elem *
find_elem (struct hash *h, struct list *bucket, struct hash_elem *e) {
  struct list_elem *i;
  for (i = list_begin (bucket); i != list_end (bucket); i = list_next (i)) {
    struct hash_elem *hi = list_elem_to_hash_elem (i);
    if (!h->less (hi, e, h->aux) && !h->less (e, hi, h->aux))
      return hi;
  }
  return NULL;
}
```
- `hash_elem`을 찾기 위해 `hash_find()` 사용하려고 한다.
- 근데, 반환 값이 `hash_elem`이고 인자로 `hash_elem`이 있어서 뭔가 이상하다 ㅡㅡ. 이 내부 구조를 알아보자
- *for문 속 `h->less`를 활용한 동등성 비교*
	- `h->less`는 hash_table_init단계에서 정의한 `hash_less_func`이다.
	- 이를 활용해서 동등성을 비교하고 있으므로 앞서 `hash_less_func`을 어떻게 구현했는지 알아보는 것이 중요.
	- 앞서 구현했던 `hash_less_func`은 **page의 첫 va주소를 기준으로 비교**하는 함수였다
	- 따라서, **va의 위치가 같은 page에 있다면** 인자로 어떤 `hash_elem`을 넣든 **찾고자 하는 page의 `hash_elem`을 찾을 수 있을 것**

---
### 1.3.  세미 완성본 
```c
bool vm_try_handle_fault(struct intr_frame *f UNUSED, void *addr UNUSED,
                         bool user UNUSED, bool write UNUSED,
                         bool not_present UNUSED) {
  struct supplemental_page_table *spt UNUSED = &thread_current()->spt;
  struct page *page = NULL;
  if (addr == NULL || is_kernel_vaddr(addr) || not_present) {
    return false;
  }

  page = spt_find_page(spt, addr);
  if (page == NULL) {
    return false;
  }

  return vm_do_claim_page(page);
}
```

```c
struct page *spt_find_page (struct supplemental_page_table *spt UNUSED, void *va UNUSED) {
  
  struct page *page = NULL;
  struct page fake_page;
  fake_page.va = pg_round_down(va);
  const struct hash_elem *h = hash_find(&spt->pages, &fake_page.hash_elem);
  if (h != NULL) {
    page = hash_entry(h, struct page, hash_elem);
  }

  return page;
}
```
- 앞서 추측했던 것처럼 `va`만 같다면 찾고자 하는 page를 얻을 수 있다. 
- 따라서 스택 내에서 만들고 사라지는 가짜 page구조체를 선언하고 여기에 인자로 넘어온 `va`를 세팅했다.

---
### 1.4.  추가 고민 
`vm_try_handle_fault`함수를 보면 `spt_find_page`를 통해 찾은 page가 `NULL`일 경우 처리를 못 하고 있다. 
`NULL`? 
```c
bool vm_try_handle_fault (struct intr_frame *f UNUSED, void *addr UNUSED,
    bool user UNUSED, bool write UNUSED, bool not_present UNUSED){
  
  ...
  page = spt_find_page(spt, addr);

  return vm_do_claim_page (page);
}  
```

SPT에 `VA`가 `miss`됐다는 뜻은❓
- SPT에 사전 등록 안되어있는 가상주소 페이지이다.
- 실행 파일의 코드/데이터 영역, BSS, `mmap`파일 등은 사전에 SPT 엔트리로 등록되어 있을 것
- 만약, 유효하고 유저 가상 주소임에도 `Page_Fault`가 났다는 것은 엔트리로 등록하지 않은 

>[!danger] swap-out된 page를 조회할 때는 SPT Hit다 
>Swap-Out된 페이지라도 SPT에 스왑위치같은 메타데이터가 적혀있을 것이다. 즉, 그 page는 hit!

따라서 Stack Growth 요건만 충족되면 SPT에 Stack용 익명 페이지를 등록하는 것이 좋다.

근데 Pintos과제 가이드 gitbook도 그렇고 가이드 영상도 그렇고 Stack Growth주제는 뒤쪽에 배치시킨걸 보면 **나중에 다뤄도 될 문제인 것 같다.**

---
### 1.5.  번외 : 최종 코드
Pintos를 마무리하고 이전 글들을 보다가 여기에도 최종 코드를 남겨보는 것도 좋을 것 같아서 간단한 설명과 함께 코드를 남겨본다.
```C
bool vm_try_handle_fault(struct intr_frame* f, void* addr, bool user,
                         bool write, bool not_present) {
  // 1. 각종 검증 
  if (!is_user_vaddr(addr)) return false;
  if (addr == NULL || !is_user_vaddr(addr) || !not_present) return false;

  struct supplemental_page_table* spt = &thread_current()->spt;
  struct page* page = NULL;
  void* upage = pg_round_down(addr);

	// 2. 페이지 찾기 
  page = spt_find_page(spt, upage);

  // 3. page가 없는 경우 처리 - stack 확장 여부 판단 및 처리 필요
  if (page == NULL) {
    return can_grow_stack(f, addr, user) ? vm_stack_growth(upage) : false;
  }

  // 4. page가 있는 경우의 예외 처리 : writable은 false인데 write시도 하는 경우
  if (!page->writable && write) {
    return false; 
  }

	// 5. page가 있는 경우 정상 처리 - fault난 page를 load요청 
  return vm_do_claim_page(page);
}
```
1. *각종 검증*
2. *페이지를 찾는다*
	- 해당 user 가상주소에 해당하는 page를 spt에서 찾는다.
	  
3. *페이지가 없는 경우 처리*
	- Stack확장이 필요한지 확인
	- 필요하면 확장, 아니면 false 반환 
	  
4. *페이지가 있는 경우 처리*
	- 예외 처리 : Page는 쓰기 불가능한데 write하려고 하면 false 반환 
	- 정상 처리 : `vm_do_claim_page`호출로 page를 메모리에 로드 요청 

---
## 2.  초기 vm_do_claim_page 세팅

Pintos에서는 `vm_do_claim_page` 를 통해 페이지 frame을 프로세스에게 할당한다.

```c
/* Claim the PAGE and set up the mmu. */
static bool vm_do_claim_page (struct page *page) {
  struct frame *frame = vm_get_frame ();
  /* page ↔ frame을 연결*/
  frame->page = page;
  page->frame = frame;

  /* TODO: Insert page table entry to map page's VA to frame's PA. */
  return swap_in (page, frame->kva);
}
```
**함수 목적** : page를 실제 메모리에 올리는 역할을 하는 함수 
1. *`vm_get_frame()`을 통해 frame을 확보*
	- But 이 함수도 구현해야하는 것이 mission
	  
2. *Page ↔ Frame을 연결*
3. *todo - PTE를 갱신한다* ⭐
	- page의 가상 주소(유저 영역)를 frame의 물리 주소에 매핑한다.(kva = 커널 가상 주소)
	- `pml4_set_page()`가 PTE를 채우는 것 
	  
4. *Swap_In 실행* 
	- 해당 페이지 타입별로 프레임 내용을 채우는 공통 진입점

---
### 2.1.  사전 개념 : 유저 VA vs 커널 VA 
참고 : [[Computer_Science/Memory_Hierarchy/커널 영역 심화\|커널 영역 심화]]

1. *User Virtual Address*
	- **각 프로세스별 독립된 주소 공간** ➡ 메모리 보호 가능
	- 프로세스 A, B가 동일한 VA라도 실제로는 서로 다른 물리 frame 과 매핑되어 있다
	  
2. *Kernel Virtual Address*
	- 커널이 쓰는 가상주소 공간
	- 보통 이 VA와 **물리 메모리를 단순 offset 기반으로 매핑**해두었음
	- 유저 모드로는 직접 접근 불가💢

>[!QUESTION] 커널 VA에 연결되어 있는데 왜 또 유저 VA에 연결시키는가❓
>- **User Process는 커널 VA에 직접 접근할 수 없다.**
>- 그래서 어떤 물리 프레임을 유저 코드/데이터가 쓰려면, 그 **같은 물리 프레임을 UVA에도 매핑**해야 한다.
>- 어떤 경우에는 memcpy도 했던 듯? in pintos


---
### 2.2.  관련 메서드 - pml4_set_page 
#페이지테이블_조작
```c
/* Adds a mapping in page map level 4 PML4 from user virtual page
 * UPAGE to the physical frame identified by kernel virtual address KPAGE.
 * UPAGE must not already be mapped. KPAGE should probably be a page obtained
 * from the user pool with palloc_get_page().
 * If WRITABLE is true, the new page is read/write;
 * otherwise it is read-only.
 * Returns true if successful, false if memory allocation
 * failed. */

// upage = 유저 가상 주소, kpage = 커널 가상 주소
bool pml4_set_page (uint64_t *pml4, void *upage, void *kpage, bool rw) {
	....

	// upage(가상주소)에 해당하는 최종 pte 위치 찾기
	uint64_t *pte = pml4e_walk (pml4, (uint64_t) upage, 1);

	// 찾은 pte에 물리 주소 + 권한 비트 기록
	if (pte)
		*pte = vtop (kpage)   // KVA ➡ 물리 주소로 변환 
				 | PTE_P 
				 | (rw ? PTE_W : 0) 
				 | PTE_U;
	return pte != NULL;
}
```
**함수 목적** 
- 4-Level page table에 **'가상 주소 ↔ 물리 주소' 매핑을 등록**하는 함수
- 커널 가상주소에서 실제 물리 주소 frame을 뽑아냄 ➡ 유저 VA와 연결시켜 줌 
- *주의 💢*
	- Pintos 기준에서는 이미 다른 물리 프레임과 mapping돼 있으면 안 된다.
	- 즉, **새 mapping만 허용**

`pml4e_walk`
- VA에 해당하는 최종 PTE 위치까지 4단계 페이지 테이블을 걸어 내려가서, 그 **PTE의 주소를 반환**하는 함수 (`PML4 → PDPT → PD → PT`)

이미 구현되어 있고 수정하지 않아도 되는 함수이므로 **대충 '어떤 역할을 하구나~' 하고 넘어가기**

`pml4_set_page()` 내에 `pml4e_walk`가 있다. 이 함수도 뭐하는지에 대해 간단히 알아볼 것 
```c
/* Adds a mapping in page map level 4 PML4 from user virtual page
 * UPAGE to the physical frame identified by kernel virtual address KPAGE.
 * UPAGE must not already be mapped. KPAGE should probably be a page obtained
 * from the user pool with palloc_get_page().
 * If WRITABLE is true, the new page is read/write;
 * otherwise it is read-only.
 * Returns true if successful, false if memory allocation
 * failed. */
bool pml4_set_page (uint64_t *pml4, void *upage, void *kpage, bool rw) {
	ASSERT (pg_ofs (upage) == 0);
	ASSERT (pg_ofs (kpage) == 0);
	ASSERT (is_user_vaddr (upage));
	ASSERT (pml4 != base_pml4);

	// upage(가상주소)에 해당하는 최종 pte 위치 찾기
	uint64_t *pte = pml4e_walk (pml4, (uint64_t) upage, 1);

	// 찾은 pte에 물리 주소 + 권한 비트 기록
	if (pte)
		*pte = vtop (kpage)   // KVA ➡ 물리 주소로 변환 
				 | PTE_P 
				 | (rw ? PTE_W : 0) 
				 | PTE_U;
	return pte != NULL;
}
```
`pml4_set_page()`
- Page Table에 **'가상 주소 ↔ 물리 주소' 매핑을 등록**하는 함수
- 즉, **유저 가상 페이지 주소(`UPAGE`)** 를 **커널 가상주소(`KPAGE`)가 가리키는 물리 프레임에 매핑**
- *주의 💢*
	- 이미 다른 물리 프레임과 mapping돼 있으면 안 된다.
	- 즉, **새 mapping만 허용**

*Args 3 - `kpage`*
- 주로 `palloc_get_page(PAL_USER)`로 할당받은 페이지일 것
- `PAL_USER` : 유조 공간용 frame을 가리켜야 한다는 뜻

---

### 2.3.  관련 메서드 -  vm_get_frame()
 초기 `vm_get_frame()`
```c
/* palloc() and get frame. If there is no available page, evict the page
 * and return it. This always return valid address. That is, if the user pool
 * memory is full, this function evicts the frame to get the available memory
 * space.*/
static struct frame *vm_get_frame (void) {
	struct frame *frame = NULL;
	/* TODO: Fill this function. */

	ASSERT (frame != NULL);
	ASSERT (frame->page == NULL);
	return frame;
}
```
- 물리 메모리에서 빈 프레임을 가져옴 
- 만약 여유 page가 없다면 다른 페이지를 evict해서 확보할 수 있음 


페이지에 실제 메모리 매핑을 할당하기 위한 `vm_do_claim_page`를 하기 위해서는 `vm_get_frame()`이라는 메서드가 필요하다. 근데, 아직 구현이 되어 있지 않은 상황

주석 曰
- palloc()을 사용해서 frame을 얻으라는데
- 가능한 page가 없으면 page를 evict해라

이러한 기능을 구현하기 위해서는 이 함수 위에 적힌 `vm_evict_frame`과 `vm_get_victim`을 구현해야 할 것 같다. 근데 frame들을 어떻게 관리할지가 제일 우선순위가 될 듯?

---
### 2.4.  Frame 테이블 관리

#### 2.4.1.  GitBook Frame Table 관리 글 
GitBook에 Frame Table 관리에 대한 글이 있다.

- 프레임 테이블에는 **각 프레임의 엔트리 정보**가 담겨 있습니다. 
- 프레임 테이블의 각 엔트리에는 현재 해당 엔트리를 차지하고 있는 페이지에 대한 포인터(있는 경우라면), 그리고 당신의 선택에 따라 넣을 수 있는 기타 데이터들이 담겨 있습니다. 
- 프레임 테이블은 **비어있는 프레임이 없을 때 쫓아낼 페이지를 골라줌**으로써, Pintos가 효율적으로 eviction policy를 구현할 수 있도록 해줍니다.
- 유저 페이지를 위해 사용된 프레임들은 `palloc_get_page(PAL_USER)`를 호출함으로써 유저 풀에서 획득된 것이어야 합니다. 
- 커널 풀에서 할당했다가 예상치 못하게 테스트 케이스에서 실패하는 일을 막기 위해서는, **반드시 `PAL_USER`를 사용**하여야 합니다.


프레임 테이블에서 가장 중요한 작업은 **사용되지 않은 프레임을 획득**하는 것입니다. 
- 이는 프레임이 `free` 상태라면 간단한 일입니다. 
- 하지만 free 상태인 프레임이 없다면, 몇몇 페이지들을 프레임에서 쫓아내어 그 프레임을 free 상태로 만들어주어야 합니다.

만약 스왑 슬롯의 할당 없이 쫓아낼 수 있는 프레임이 없는데, 스왑 슬롯마저 꽉 차 있다면, 커널을 패닉시킵니다. 실제 OS들은 이런 상황을 막거나 복구하기 위해 다양한 정책들을 적용하고 있습니다만, 그러한 정책들은 이 프로젝트의 범위를 벗어납니다.

eviction 절차는 대략 다음 단계들로 구성됩니다 :
1. 당신의 페이지 재배치 알고리즘을 이용하여, 쫓아낼 프레임을 고릅니다. 아래에서 설명할 “accessed”, “dirty” 비트들(페이지 테이블에 있는)이 유용할 것입니다.
2. 해당 프레임을 참조하는 모든 페이지 테이블에서 참조를 제거합니다. 공유를 구현하지 않았을 경우, 해당 프레임을 참조하는 페이지는 항상 한 개만 존재해야 합니다.
3. 필요하다면, 페이지를 파일 시스템이나 스왑에 write 합니다. 쫓아내어진(evicted) 프레임은 이제 다른 페이지를 저장하는 데에 사용할 수 있습니다. 

---
#### 2.4.2.  초기 고민 🤔

>[!QUESTION] `frame` 테이블은 어떤 자료구조가 좋을까?

```C
struct frame {
		void *kva;  // 프레임이 매핑된 커널 VA
		struct page *page; // 이 프레임을 점유하고 있는 User Page
}
```

>[!danger] frame 자체가 실제 물리 메모리 페이지가 아님 
>물리 메모리의 한 페이지와 관리 정보들(ex. 누가 점유 중, 어떤 `KVA`랑 연결 중 등)을 추상화한 구조체 
>- 유저 `VA`를 가진 `page`를 가진 이유 : 유저에 이 `frame`을 매핑한다  = **'유저 `VA` -> 커널 `VA` -> 실제 메모리'로 매핑되게** 한다. ⭐⭐

`frame_table`에
- 사용 중인 `frame`들을 `list`로 관리 : LRU? 알고리즘 사용 시 앞에서?뒤에서부터 빼면 됨
- 고민해야 될 점은 '과연 사용하고 있지 않은 `frame`들도 관리를 해야하냐?' 는 것이다.

 여러 page들이 공유하는 `frame`인지 아닌지도 고민해야 할 듯?

그럼 처음 frame들은 도대체 어디에 있지? page_init인가 그런 곳에서 커널 VA랑 매핑해줬던 것 같은데... 아 `paging_init`이네 
실제 메모리 로딩은 `swap_in()` 내부에서 함 
그럼 이제 `Swapping` 공부해볼까??

Swapping 관련 링크 : [[Pintos/Pintos VM/Pintos VM - Swapping\|Pintos VM - Swapping]]

#### 2.4.3.  실제 관리 및 구현
링크 : [[Pintos/Pintos VM/Pintos VM - Frame 관리와 알고리즘\|Pintos VM - Frame 관리와 알고리즘]]
