---
{"dg-publish":true,"permalink":"/Pintos/Pintos VM/Pintos VM - 초기 부분 정리/","noteIcon":"","created":"2025-12-03T14:52:52.686+09:00","updated":"2025-12-13T09:26:33.447+09:00"}
---


### 0.1.  목차

[[#SPT 초기화 시키기|SPT 초기화 시키기]]
- [[#SPT 초기화 시키기#1.   supplemental_page_table_init 후기|1.   supplemental_page_table_init 후기]]
	- [[#1.   supplemental_page_table_init 후기#1.1.  Hash 자료구조 선택|1.1.  Hash 자료구조 선택]]
	- [[#1.   supplemental_page_table_init 후기#1.2.  hash_init|1.2.  hash_init]]
	- [[#1.   supplemental_page_table_init 후기#1.3.  list_less_func|1.3.  list_less_func]]
	- [[#1.   supplemental_page_table_init 후기#1.4.  hash_hash_func - 해시값 생성 함수|1.4.  hash_hash_func - 해시값 생성 함수]]
	- [[#1.   supplemental_page_table_init 후기#1.5.  SPT 초기화 과정 전체 정리|1.5.  SPT 초기화 과정 전체 정리]]

[[#vm_init 관련 로직들|vm_init 관련 로직들]]
- [[#vm_init 관련 로직들#1.  vm_annon_init|1.  vm_annon_init]]
- [[#vm_init 관련 로직들#2.  vm_alloc_page_with_initializer - 페이지 생성 예약|2.  vm_alloc_page_with_initializer - 페이지 생성 예약]]
	- [[#2.  vm_alloc_page_with_initializer - 페이지 생성 예약#2.1.  무엇인가?|2.1.  무엇인가?]]
	- [[#2.  vm_alloc_page_with_initializer - 페이지 생성 예약#2.2.  코드 완성 및 분석|2.2.  코드 완성 및 분석]]
		- [[#2.2.  코드 완성 및 분석#2.2.1.  spt_find_page - 중복 체크|2.2.1.  spt_find_page - 중복 체크]]
		- [[#2.2.  코드 완성 및 분석#2.2.2.  page 구조체 생성|2.2.2.  page 구조체 생성]]
		- [[#2.2.  코드 완성 및 분석#2.2.3.  페이지 타입별 초기화 함수 선택|2.2.3.  페이지 타입별 초기화 함수 선택]]
		- [[#2.2.  코드 완성 및 분석#2.2.4.  uninit 페이지로 초기화|2.2.4.  uninit 페이지로 초기화]]
	- [[#2.  vm_alloc_page_with_initializer - 페이지 생성 예약#2.3.  어디서 사용되는가?|2.3.  어디서 사용되는가?]]
	- [[#2.  vm_alloc_page_with_initializer - 페이지 생성 예약#2.4.  관련 메서드 - lazy_load_segment|2.4.  관련 메서드 - lazy_load_segment]]
		- [[#2.4.  관련 메서드 - lazy_load_segment#2.4.1.  초기 코드|2.4.1.  초기 코드]]
		- [[#2.4.  관련 메서드 - lazy_load_segment#2.4.2.  추측|2.4.2.  추측]]
		- [[#2.4.  관련 메서드 - lazy_load_segment#2.4.3.  todo 완성(최종본은 아님)|2.4.3.  todo 완성(최종본은 아님)]]
		- [[#2.4.  관련 메서드 - lazy_load_segment#2.4.4.  최종본|2.4.4.  최종본]]
	- [[#2.  vm_alloc_page_with_initializer - 페이지 생성 예약#2.5.  관련 메서드 - annon_initializer|2.5.  관련 메서드 - annon_initializer]]
		- [[#2.5.  관련 메서드 - annon_initializer#2.5.1.  개념|2.5.1.  개념]]
		- [[#2.5.  관련 메서드 - annon_initializer#2.5.2.  초기 세팅 및 추리|2.5.2.  초기 세팅 및 추리]]
		- [[#2.5.  관련 메서드 - annon_initializer#2.5.3.  완성본|2.5.3.  완성본]]
	- [[#2.  vm_alloc_page_with_initializer - 페이지 생성 예약#2.6.  관련 메서드 - file_backed_initializer|2.6.  관련 메서드 - file_backed_initializer]]
- [[#vm_init 관련 로직들#3.  setup stack|3.  setup stack]]


Pintos를 시작한지 벌써 4주차이다. 4주차 부터는 Virtual Memory라는 키워드를 중심으로 OS를 설계하는 작업을 한다.
Project 1, 2였던 thread, User Program에서도 마찬가지지만 초반에는 항상 어려웠다. 이유는 1) 프로젝트 요구사항 분석 2) 관련 파일들이 뭐가 있는지 찾기 3) 관련 파일들을 분석 등을 해야하기 때문에 길을 찾는과정이 필요하다. Project 1,2에 비해서 Project 3인 VM은 함수, 메크로, 파일들이 너무 많아서 초반에 시간이 꽤 걸렸다.

참고로 이 글은 최종본이랑 코드가 다를 수 있다. 왜냐면 코드는 계속 미션을 수행할 때마다 변하기 때문이다.
최종 코드는 GitHub를 참고하거나 다른 글들을 보면 좋을 것 같다.


---
# SPT 초기화 시키기 

---
## 1.   supplemental_page_table_init 후기
SPT 개념 : [[Pintos/Pintos VM/Pintos VM - 구조체 및 관련 개념 정리\|Pintos VM - 구조체 및 관련 개념 정리]] - Page vs SPT 참고 

> 각 프로세스(스레드)의 페이지 정보를 저장하는 보조 테이블을 사용하기 위해 세팅하는 단계

---
### 1.1.  Hash 자료구조 선택

 supplemental_page_table을 Hash 자료구조로 사용하고 싶었다.
 이유는 여러 가지가 있지만 주 이유는 '**빠른 검색 속도**' 이다.
 - 해쉬 자료구조는 충돌만 안나면 O(1)로 매우 빠른 검색이 가능하다.
 - Lazy Load기반의 OS에서는 Page Fault나는 경우가 많기 때문에 해당 page 복구를 위해 빠르게 검색하는 것이 중요하다. 따라서 Hash 자료구조 선택

---
### 1.2.  hash_init
SPT를 처음 생성할 떄, 내부 hash구조체(여기서는 hash_table)를 초기화한다.
구조체의 초기 정의는 다음과 같다.
```c
struct hash {
  size_t elem_cnt;            /* Number of elements in table. */
  size_t bucket_cnt;          /* Number of buckets, a power of 2. */
  struct list *buckets;       /* Array of `bucket_cnt' lists. */
  // ✅구현해야할 함수 1
  hash_hash_func *hash;       /* Hash function. */
  // ✅구현해야할 함수 2
  hash_less_func *less;       /* Comparison function. */
  void *aux;                  /* Auxiliary data for `hash' and `less'. */
};
```
`struct list *buckets`가 있는 이유 - 해시 충돌 처리 
- 동일한 해시값이 발생할 경우를 대비해 해당 Bucket은 내부적으로 연결 리스트로 관리된다.
- 이를 구현한 것이 `list *buckets`이다.
- 이를 통해 해시 충돌이 발생하더라도 데이터 손실 없이 저장 및 조회가 가능하다.
- 참고 - java에서의 최적화❓
	- 자바도 hash자료구조를 사용하는데 해시 충돌 발생 시 성능저하를 최소화하기 위해 Red-Black Tree를 사용한다 (리스트 -> R.B Tree : $O(n) -> O(log{n})$)

 **`hash_init()` 함수 실행**
 - Pintos에서 자료구조(ex. list, hash, bitmap)를 사용하기 위해서는 우선 실행을 시켜야한다. 그 함수는 스켈레톤 코드로 주어지면 hash자료구조는 `hash_init()`이라는 함수가 주어지낟.
 - 그 함수는 아래와 같다.
	 ```c
	 bool hash_init (struct hash *h, hash_hash_func *hash, hash_less_func *less, void *aux)
	 ```
	- 보면 인자에 `function` 2개가 필요한 것을 볼 수 있다. 이 2가지 함수를 구현하고 `hash_init`을 이용하면 될 것이다.
	- 이 함수 구현 시 막연하게 구현할 필요는 없고 Pintos에서 제공하는 범용 해시 함수들을 쓰면 좋다(ex. `hash_entry`, `hash_bytes`)

---
### 1.3.  list_less_func
> **해시 충돌 시** 정렬 or 비교 기준을 제공하는 함수이다

```c
/* hash_elem을 va로 비교하는함수 */

bool page_less(const struct hash_elem *a, const struct hash_elem *b, void *aux) {
  struct page *page1 = hash_entry(a, struct page, hash_elem);
  struct page *page2 = hash_entry(b, struct page, hash_elem);
  if (page1 == NULL || page2 == NULL) {
    return false;
  }

  return page1->va < page2->va ? true : false;
  /* A가 B보다 작으면 true를 반환하고,
   * A가 B보다 크거나 같으면 false를 반환한다. */
}
```
Project 1에서 `priority`의 `less`함수를 구현했을 때처럼 하면 된다.

>[!tip] 항상 page 경계로 비교를 하는 것이 좋다
>- 주소 비교 시 단순한 byte가 아닌 페이지 단위로 비교하는 것이 좋다.
>- Cuz 그렇게 해야 **같은 페이지 안의 서로 다른 byte주소도 같은 페이지를 가리킬 수** 있다.
>- But **`hash_entry`는 항상 struct page 객체의 시작 주소를 가리킴** ⭐ ➡ 따라서, **굳이 조정 안해도 페이지 단위 정렬이 보장**된다.
>	- 내부를 보면 `offsetof`를 통해 항상 페이지 앞의 시작 주소를 가리키도록 함 

---
### 1.4.  hash_hash_func - 해시값 생성 함수
> - 해시 함수는 특정 값을 기반으로 고유한 해시값을 생성한다.
> - 그 값의 기준을 정하고 고유한 해시 값을 생성하는 것이 미션

Pintos에서 제공하는 범용 해시 함수들을 쓰면 좋다(ex. `hash_entry`, `hash_bytes`)
```c
/* Computes and returns the hash value for hash element E, given
 * auxiliary data AUX. */
typedef uint64_t hash_hash_func (const struct hash_elem *e, void *aux);

// 구현 
static uint64_t hash_hash_func (const struct hash_elem *e, void *aux){
  const struct page *p = hash_entry(e, struct page, hash_elem);
  return (uint64_t) hash_bytes(&p->va, sizeof p->va);
}
```

`hash_bytes(void *buf, size_t)`
- Pintos에서 제공하는 범용 해시 함수 
- 이 함수는 **메모리 버퍼의 내용이 아니라 그 주소 값 자체를 기반으로 해싱**한다.
- 주의 : 그 주소의 내용이 아니라 **주소 값 자체를 넘겨야** 한다.(Cuz 주소 기반 해쉬 함수)

Pintos에서는 `vm_do_claim_page` 를 통해 페이지 `frame`을 프로세스에게 할당한다.

---
### 1.5.  SPT 초기화 과정 전체 정리
> 위의 과정을 다 했다면 해쉬 형태의 SPT 초기화는 아래와 같다.

```c
void supplemental_page_table_init(struct supplemental_page_table* spt UNUSED) {
  ASSERT(spt != NULL);
  hash_init(&spt->page_table, page_hash, page_less, NULL);
}
```
이렇게 되면 각 bucket 내부는 `page_less_func()` 기준으로 정렬되며,  탐색 시 `page_hash_func()`으로 bucket을 찾은 뒤, 해당 리스트에서 비교를 수행한다.

---
# vm_init 관련 로직들

Pinto에서는 main 프로그램이 첫 부팅 시 여러 작업들을 한다.
그 중 Project 3부터는 `vm_init`이란 것을 해야하는데 그것을 완성해보자
```c
int main (void) {

	....
#ifdef VM
  vm_init (); // ⬅요거
#endif

  printf ("Boot complete.\n");
  ....
  thread_exit ();
}	
```
```c
/* Initializes the virtual memory subsystem by invoking each subsystem's
 * intialize codes. */
void vm_init(void) {
  vm_anon_init();
  vm_file_init();
#ifdef EFILESYS /* For project 4 */
  pagecache_init();
#endif
  register_inspect_intr();
  /* DO NOT MODIFY UPPER LINES. */
  list_init(&frame_table);
  lock_init(&frame_lock);
  // clock_ptr = list_begin(&frame_table);  // 이건 나중에 SWAP관리 시 쓰는거랑 나중에 설명 
}
```
우선 딱봐도 Project 3에서 건드려야 할 것은 `vm_annon_init`과 `vm_file_init`같아 보인다.

---
## 1.  vm_annon_init
익명페이지를 init시키는 함수같다.
GitBook 가이드 분석을 해보자 - [[Pintos/Pintos VM/가이드 및 번역본/VM 익명 페이지 - GitBook 가이드 해석본 및 분석\|VM 익명 페이지 - GitBook 가이드 해석본 및 분석]]

음... 일단은 todo도 안 적혀있고 아직은 구현해야 할 필요를 못 느꼈다
```c
static struct disk *swap_disk;

/* Initialize the data for anonymous pages */
void vm_anon_init (void) {
  swap_disk = disk_get (1, 1);
}
```
- `disk_get (1, 1)` 
	- 스왑용 블록 장치를 가져와서 swap in/out을 할 물리 저장소를 지정하는 초기화 
	  
- `swap_disk`
	- 익명 페이지를 메모리에서 쫓아낼 때 내용을 임시로 저장/복구할 swap 장치 핸들 

>[!QUESTION] 왜 disk초기화를 하는가?
>- 익명 페이지는 file-backed 페이지와 달리 백업본이 없다.
>- 즉, **메모리가 부족으로 page-out시 어딘가에 원본 저장 必 ➡Swap Disk**

---
## 2.  vm_alloc_page_with_initializer - 페이지 생성 예약
```c
bool vm_alloc_page_with_initializer (enum vm_type type, void *upage, bool writable, vm_initializer *init, void *aux) { // init - 초기화 함수, aux - 보조 데이터 
	...  
  struct supplemental_page_table *spt = &thread_current ()->spt;
  if (spt_find_page(spt, upage) == NULL) {
    /* TODO: Create the page, fetch the initialier according to the VM type,
     * TODO: and then create "uninit" page struct by calling uninit_new. You
     * TODO: should modify the field after calling the uninit_new. */

    /* TODO: Insert the page into the spt. */
  }
  ...
```

---
### 2.1.  무엇인가?
> 페이지를 생성하고 초기화하는 **과정을 예약**하는 함수


- 단순히 페이지를 즉시 생성하는 것이 아니라, 이 페이지가 나중에 필요해질 때 어떻게 로드할지를 미리 등록하는 함수이다.
- 물리 메모리 즉시 할당❌ ➡ 지연 로드 ✅
- 페이지가 실제로 접근되어 **Page Fault가 발생할 때 등록해둔 초기화 함수를 실행시켜 물리 Frame을 할당**

이 방식이 바로 `Lazy Loading`의 핵심이다.

---
### 2.2.  코드 완성 및 분석
```c
/* Create the pending page object with initializer. If you want to create a
 * page, do not create it directly and make it through this function or
 * `vm_alloc_page`. */
bool vm_alloc_page_with_initializer(enum vm_type type, void* upage,
                                    bool writable, vm_initializer* init,
                                    void* aux) {
	// 1. 현재 스레드의 SPT를 가져옴                                    
  struct supplemental_page_table* spt = &thread_current()->spt;
  
  // 2. 특정 가상주소에 매핑된 Page가 SPT에 등록되어 있는지 확인
  if (spt_find_page(spt, upage) == NULL) {
	  // 3. 새로운 page 객체 생성
    struct page* page = malloc(sizeof(struct page));
    if (page == NULL) {
      goto err;
    }
    /* 페이지의 가상주소와 권한을 할당 */
    page->va = upage;
    page->writable = writable;

		// 4. 페이지 타입별 초기화 로직 선택
    bool (*initializer)(struct page*, enum vm_type, void* kva) = NULL;
    switch (VM_TYPE(type)) {
      case VM_ANON:
        initializer = anon_initializer;
        break;
      case VM_FILE:
        initializer = file_backed_initializer;
        break;

      default:
        free(page);
        goto err;
    }

    // 5. Uninitialzed Page` 객체 생성
    uninit_new(page, upage, init, type, aux, initializer);
    page->writable = writable;

		// 6. SPT에 등록 
    if (!spt_insert_page(spt, page)) {
      free(page);
      goto err;
    }
  }
  /* 성공했으면 true 반환 */
  return true;

err:
  return false;
}
```
*전체적인 함수 동작 흐름* 
1. 현재 스레드의 SPT를 가져옴
2. 특정 가상주소에 매핑된 Page가 SPT에 등록되어 있는지 확인 - `spt_find_page`
	- 있으면 false
	- 없다면 3번 진행
3. 새로운 page 객체 생성 - `malloc()`
4. 페이지 타입별 초기화 로직 선택 - `initializer`
5. `Uninitialzed Page` 객체 생성
6. SPT에 등록 

---
#### 2.2.1.  spt_find_page - 중복 체크 
```c
if (spt_find_page (spt, upage) == NULL) {
// 이미 같은 VA가 등록되어 있으면 중복 생성 방지 
```
- SPT에 인자로 넘어온 **가상주소에 해당하는 page가 등록되어 있지 않을 때에만 진행**하는 로직
- *의도* : `upage`(유저 가상주소)에 이미 페이지 메타가 등록되어 있으면 중복 등록을 안하려는 의도

---
#### 2.2.2.  page 구조체 생성 
```c
  struct page *page = malloc (sizeof *page);
  if (page == NULL)
    goto err;
	page->va = upage;
	page->writable = writable;
```
- SPT에 등록하기 위한 페이지 메타 구조체 동적할당 
- 여기서 만들어지는 `struct page`는 실제로 메모리에 올라간 페이지가 아니라,  **페이지의 “메타데이터”** (향후 로딩을 위한 정보만 저장하는 껍데기)다.
  
>[!QUESTION] 왜 palloc_get_page()가 아닌 malloc을 사용했는가?
>- SPT의 엔트리로 사용되는 **page는 메타데이터**이다.
>- 이는 user가 직접 접근하는 개념이 아님 unlike `frame`'
>- 즉, 이 page 구조체는 아래와 같은 이유로 `malloc()`이 적합 
>	1. 커널 내부에서만 쓰이는 메타데이터 
>	2. 진짜 메모리 공간이 필요한게 아님 
>- palloc으로 해도 되는데 낭비가 심함
>	- 강제 4KB
>	- USER/KERNEL Pool같은 물리 메모리 자원 먹음 

---
#### 2.2.3.  페이지 타입별 초기화 함수 선택 
```C
		// 반환 타입이 bool인 함수 포인터 (initializer)
    bool (*initializer) (struct page *, enum vm_type, void *kva) = NULL;
    switch (VM_TYPE (type)) {
	    // 타입별 함수 주소를 젖장 
      case VM_ANON:
        initializer = anon_initializer;
        break;
      case VM_FILE:
        initializer = file_backed_initializer;
```
- 타입별로 함수 포인터를 만든다.
- 타입별로 어떤 함수를 써야할지 달라지므로 `switch`문 사용하여 원하는 함수를 담고 나중에 호출하게 하는 구조
- 여기서 선택된 **`initializer`는 실제 페이지가 로드될 때 호출된다.**

---
#### 2.2.4.  uninit 페이지로 초기화 
> 핵심 - 나중에 Page Fault 시 `uninit_initialize()`를 통해 등록된 `initializer`를 호출한다.

```C
uninit_new (page, upage, init, type, aux, initializer);
```
- 아직 frame을 배정하지 않은 page를 `uninit`상태로 초기화하는 것
- 즉, `uninit page`는 당장 물리 `frame`을 배정 받지 못한 페이지
- 대신, 나중에 PF 시 실행할 초기화 정보들을 page에 담는 함수이다.
```C
void uninit_new (struct page *page, void *va, vm_initializer *init, enum vm_type type, void *aux,
    bool (*initializer)(struct page *, enum vm_type, void *)) {

	...
	// page : 결과를 써 넣을 페이지 메타 데이터 
  *page = (struct page) {
    .operations = &uninit_ops,
    .va = va,  // 이 페이지가 담당할 유저 가상주소 
    .frame = NULL, /* no frame for now */
    .uninit = (struct uninit_page) {
      .init = init,  // 상위 초기화기 
      .type = type, // 페이지 타입 
      .aux = aux,  // init이 사용할 부가 데이터 (ex. offset)
      .page_initializer = initializer, // 타입별 저수준 초기화기 
    }
```
`initializer`
- initalizer를 통해 **fram이 실제로 할당되면** 해당 타입에 맞춰 **page 내부/frame을 세팅**하는 함수 
- 페이지의 정체성을 결정
- `page->operations`·내부 필드들을 **해당 타입(ANON/FILE 등)** 으로 바꿔 세팅하는 역할.

*`init` - 실제 프레임에 내용을 채우는 로딩 작업 수행 *
- 상위 초기화기이다.
- Pintos에서는 `lazy_load_segment`함수를 init초기화기로 쓰고 있다.
	- `load_segment()`에서 각 페이지를 `vm_alloc_page_with_initializer`로 등록할 때, `init` 인자로 `lazy_load_segment`를 넘겨줌.
- `lazy_load_segment`함수를 만드는 것도 必
```c
/* From here, codes will be used after project 3.
 * If you want to implement the function for only project 2, implement it on the
 * upper block. */
static bool lazy_load_segment (struct page *page, void *aux) {
  /* TODO: Load the segment from the file *
  /* TODO: This called when the first page fault occurs on address VA. */
  /* TODO: VA is available when calling this function. */
}

...
static bool load_segment (struct file *file, off_t ofs, uint8_t *upage,
    uint32_t read_bytes, uint32_t zero_bytes, bool writable) {
	
    if (!vm_alloc_page_with_initializer (VM_ANON, upage, writable, lazy_load_segment, aux)) ✅
      return false;    
```


---
### 2.3.  어디서 사용되는가?
> Pintos 내 필요할 때만 페이지를 실제로 만들도록 예약하는 공통 인터페이스

Pintos를 하다 보면 다양한 곳에서 사용된다.
1. ELF 로딩 과정 속 `load_segment()`안 
	- **ELF Lazy 로딩** : 커널이 실행 파일 ELF를 읽을 때, 모든 코드/데이터를 한 번에 메모리에 올리지 않음 
	- 실행 파일의 각 segment(코드, 데이터)는 **물리 메모리를 즉시 사용하지 않고,**
	- **`PF`가 났을 때** 3번째 인자인 **`vm_initializer`가 실행되어 해당 부분만 Disk에서 읽어 메모리에 적재**
	  
2. `stack_growth` 시 
	- 유저 스택은 실행 중에 **필요할 때마다 확장**된다.
	- 스택 확장이 필요하면 `vm_alloc_page_with_initializer()`를 호출해 새 페이지를 **익명(anonymous) 페이지로 예약**하고 실제 접근 시 물리 페이지 할당 
	  
3. `mmap` 시 
	- 파일을 메모리에 매핑할 떄 파일의 각 페이지를 `vm_alloc_page_with_initializer()`로 등록한다
	- 이것도 Lazy-Load원리 이용 
	  
4. `page_copy` 시 (`fork`)
	- 프로세스 복제(`process_fork()`) 시, 부모의 SPT를 순회하며 각 페이지를`vm_alloc_page_with_initializer()`로  자식 프로세스 SPT에 동일하게 등록


---
### 2.4.  관련 메서드 - lazy_load_segment
---
#### 2.4.1.  초기 코드 
```c
/* From here, codes will be used after project 3.
 * If you want to implement the function for only project 2, implement it on the
 * upper block. */
static bool lazy_load_segment (struct page *page, void *aux) {
  /* TODO: Load the segment from the file *
  /* TODO: This called when the first page fault occurs on address VA. */
  /* TODO: VA is available when calling this function. */
  // TODO가 많다.
}
```
*뭐하는 함수❓*
-  위에서 `vm_alloc_page_with_initializer`를 보면서 `init`함수가 적힌 곳을 보면 알 수 있듯이 이 함수는 **`PF`가 났을 때 file의 `segment`를 `page`에 `load`하는 함수**이다.

---
#### 2.4.2.  추측
- `load`하기 위해 뭐를 할지는 Project 2까지 있던 `load` 함수를 보면서 힌트를 얻을 수 있을 것 같다.(결국 늦게 `load`하나 즉시 `load`하나 흐름은 거의 똑같다고 생각)
- 인자 `aux`를 사용하는지 아닌지 먼저 생각해봐야 한다.(함수의 인자가 의미 없을 가능성은 적음)
	- 실제 넘겨주기 전에 `aux`가 뭔지를 보니 todo가 적혀있다. 실제로 구현해야 하는 것 같다
		```c
		/* TODO: Set up aux to pass information to the lazy_load_segment. */
		void *aux = NULL;
		if (!vm_alloc_page_with_initializer (VM_ANON, upage,
					writable, lazy_load_segment, aux))
			return false;
		```
	- *TODO* : Set up aux to pass information to the lazy_load_segment
		- `aux`를 set up 하라고 한다.
		- 결국 `lazy_load_segment`함수를 구현하기 전에 인자로 넘어온 `aux`를 설계해야 한다. (Cuz `load_segment`함수에서 넘겨)
		- 무슨 정보를 줘야하지...??
- Pintos에서 너무 막연할 때는 이미 선언된 구조체 or 함수가 있는지 등을 찾아보는 것이 중요하다.
- `lazy_load_segment`함수가 있는 `process.c`파일 상단에 `aux`전용 구조체처럼 보이는 `lazy_load_aux`가 보인다. 이거를 이용하면 될 것 같다.
	```c
	#ifdef VM
	struct lazy_load_aux {
	  struct file *file; //읽어올 파일
	  off_t ofs; // ofsfset 
	  size_t page_read_bytes; // 이번 page에서 파일에서 읽어 채울 byte 수
	  size_t page_zero_bytes; // 이번 page에서 읽어온 byte를 제외하고는 zero로 채워야하는데 그거 byte 수 
	};
	```

file?, read_bytes? 등을 보면서 '뭐지? 처음이니까 다 0으로 채우면 되나'라고 생각했다.
하지만 `aux`를 정의하는 `load_segment`를 다시 보니 관련 정보들이 많다는 것을 봤고 이거를 활용하면 끝날듯?
```c
static bool
load_segment (struct file *file, off_t ofs, uint8_t *upage,
		uint32_t read_bytes, uint32_t zero_bytes, bool writable) {

	while (read_bytes > 0 || zero_bytes > 0) {
		/* Do calculate how to fill this page.
		 * We will read PAGE_READ_BYTES bytes from FILE
		 * and zero the final PAGE_ZERO_BYTES bytes. */
		size_t page_read_bytes = read_bytes < PGSIZE ? read_bytes : PGSIZE;
		size_t page_zero_bytes = PGSIZE - page_read_bytes;

		/* TODO: Set up aux to pass information to the lazy_load_segment. */
		void *aux = NULL;
		if (!vm_alloc_page_with_initializer (VM_ANON, upage,
					writable, lazy_load_segment, aux))
			return false;

		/* Advance. */
		read_bytes -= page_read_bytes;
		zero_bytes -= page_zero_bytes;
		upage += PGSIZE;
	}
```

---
#### 2.4.3.  todo 완성(최종본은 아님)
> 이 코드는 최종본은 아니다. 애초에 초기 VM구현에서는 그렇게 상세하게 설계할 필요가 없기 때문이다.(아직 file-backed도 제대로 구현되지 않은 상태이기에) 최종본은 2.4.4에 넣겠다.

*☑aux 채우기*
```c
    struct lazy_load_aux *aux = malloc(sizeof(struct lazy_load_aux));
    if (aux == NULL){
      return false;
    }
    aux->file = file;
    aux->ofs = ofs;
    aux->page_read_bytes = page_read_bytes;
    aux->page_zero_bytes = page_zero_bytes;
```
- `malloc` 사용
	- PF가 나면 `load_segment`가 아니라 `lazy_load_segment`가 실행되기 때문에 스택 변수가 아니라 heap 변수로 할당해야 한다
	- 대신, 해제는 `lazy_load_segment` 내에서 해야 함 

*☑lazy_load_segment*
- 참고 1. 이건 앞서 추측해서 말했듯이 먼저 project 1,2 에서 구현되어 있던 `load`함수를 참고해봐야 할 듯
- 참고 2. GitBook


**기존 `Load`에서 `page`에 메모리를 `load`하는 로직**은 아래와 같다
```c
/* Loads a segment starting at offset OFS in FILE at address UPAGE.
 * In total, READ_BYTES + ZERO_BYTES bytes of virtual memory are initialized, as follows:
 *
 * - READ_BYTES bytes at UPAGE must be read from FILE
 * starting at offset OFS.
 *
 * - ZERO_BYTES bytes at UPAGE + READ_BYTES must be zeroed.
 *
 * The pages initialized by this function must be writable by the
 * user process if WRITABLE is true, read-only otherwise.
 *
 * Return true if successful, false if a memory allocation error
 * or disk read error occurs. */
// 💢 기존 load_segment의 일부 
static bool load_segment() {

		// ............
    /* Get a page of memory. */
    uint8_t *kpage = palloc_get_page (PAL_USER);
    if (kpage == NULL)
      return false;

    /* Load this page. */
    if (file_read (file, kpage, page_read_bytes) != (int) page_read_bytes) {
      palloc_free_page (kpage);
      return false;
    }
    
    // 읽고 남은 부분 0으로 채우기 
    memset (kpage + page_read_bytes, 0, page_zero_bytes);
    
    /* Add the page to the process's address space. */
    // 페이지 매핑
    if (!install_page (upage, kpage, writable)) {
      printf("fail\n");
      palloc_free_page (kpage);
      return false;
    }
	 ....   
}
```
흐름을 보자
1. 유저 영역에서 페이지 가져오고 커널 가상주소로 사용
	- `PAL_USER` 플래그 : **유저 영역(user pool)  페이지를 가져와라”** 라는 뜻.
		- 페이지 풀은 크게 2가지로 나뉨
		- 1. `Kernel Pool` : 커널이 내부적으로 쓰는 자료구조, 버퍼 등
		- 2. `User Pool` : 프로세스 주소 공간(user space)에 매핑할 수 있는 페이지들. ``
		```c
		/* Two pools: one for kernel data, one for user pages. */
		static struct pool kernel_pool, user_pool;
		```
	- `palloc_get_page (PAL_USER)` : User Program의 메모리로 쓸 수 있는 page달라는 거
	  ![Pasted image 20250929153406.png](/img/user/supporter/image/Pasted%20image%2020250929153406.png)
		- *이미 매핑되어 있는데 달라하는 이유* : frame에 매핑되어 있는 page 자체를 누가 소유, 어떤 프로세스가 소유하는지는 적어야 되므로 

>[!tip] 부팅 시 매핑 전약 - Pintos vs 실제 
>1. *Pintos*
>	- 부팅 시점에 물리 메모리를 통째로 커널 주소 공간에 1:1 매핑
>	- 즉, **커널 가상 주소는 이미 매핑된 영역**
>	- 따라서 `palloc_get_page()`로 page 요청하는 것은 이미 실제 frame과 매핑된 페이지 주소를 주는 것
>	  
>2. *실제*
>	- **일부 지연 매핑** : 커널 가상주소 중 유저 영역 전용 가상주소는 지연 매핑 
>	- 이 외에도 **복잡한 정책으로 최적화** 있는데 이건 pass



2. `file_read(file, kva, size)` - 파일을 읽은 뒤 커널 VA에 채우기 
	- **커널 VA를 활용해 메모리를 Load**한다.
	- *❓파일 Load는 메모리에 하는건데 가상 주소가 왜 필요❓*
		- Pintos에서는 부팅 시 물리 `RAM`을 커널 `VA`에 매핑해둔 상태이다.
		- 따라서 커널 `VA`에 쓰기만해도 `MMU`가 알아서 해당 풀리 RAM(frame)에 기록함
		  
3. `memset` - 잔여 영역 0으로 채우기
4. `install_page` - 유저 VA 페이지와 커널 가상주소 PAGE 매핑


흠... 대충 흐름은 다 알겠는데 VM이 define될 때는 `install_page`도 없네? 좀 더 힌트를 얻어야 할 것 같은데 GitBook을 보자. GitBook에 적힌 내용은 다음과 같다.

```c
static bool lazy_load_segment (struct page *page, void *aux);
```
```text
이 함수는 실행 가능한 파일의 페이지들을 초기화하는 함수이고 page fault가 발생할 때 호출됩니다. 
이 함수는 페이지 구조체와 aux를 인자로 받습니다. 
aux는 load_segment에서 당신이 설정하는 정보입니다. 
이 정보를 사용하여 당신은 세그먼트를 읽을 파일을 찾고 최종적으로는 세그먼트를 메모리에서 읽어야 합니다.
```

---
#### 2.4.4.  최종본 
> 이 최종 코드는 VM 프로젝트가 끝난 후 구현되어 있던 코드이다. 답을 바로 보는 것은 없으나 참고차 올리는 코드

```C
bool lazy_load_segment(struct page* page, void* aux) {
  struct lazy_load_aux* llaux = (struct lazy_load_aux*)aux;
  struct file_page* file_page = &page->file;
  
  file_page->file = llaux->file;
  file_page->ofs = llaux->ofs;
  file_page->page_read_bytes = llaux->page_read_bytes;
  file_page->page_zero_bytes = llaux->page_zero_bytes;
  file_page->is_writable = llaux->is_writable;

  void* kva = page->frame->kva;
  off_t read_bytes = file_read_at(
      llaux->file, kva, (off_t)llaux->page_read_bytes, (off_t)llaux->ofs);

  if (read_bytes != llaux->page_read_bytes) {
    if (llaux->is_reopened) file_close(llaux->file);
    free(aux);
    return false;
  }

  page->writable = llaux->is_writable;
  memset(kva + read_bytes, 0, llaux->page_zero_bytes);
  free(aux);
  return true;
}
```
- 최종본은 file-backed기반 로직, copy관련 로직이 완성된 상태라 추가된게 있음 

---
### 2.5.  관련 메서드 - annon_initializer

---
#### 2.5.1.  개념 
참고 : [[Pintos/Pintos VM/Pintos VM - initializer들만 따로 보기\|Pintos VM - initializer들만 따로 보기]]

---
#### 2.5.2.  초기 세팅 및 추리 
초기에 세팅된 `anon_initializer`는 아래와 같다
```c
/* Initialize the file mapping */

bool anon_initializer(struct page *page, enum vm_type type, void *kva) {
  /* Set up the handler */
  page->operations = &anon_ops;
  struct anon_page *anon_page = &page->anon;  // 요놈 있는 이유가 있을텐뎅...
}
```
*추리*
- 일단 초기에는 `anon_page` 구조체는 텅 비어있는 상태인데 이게 세팅되어 있는 것을 보아 `anon_page` 구조체도 바꿔야 할 듯? 그리고 kva가 넘어온게... 요것도 사용해야 할 듯 
>[!QUESTION] 무슨 필드를 추가해야할까❓
>- `diskc.c`관련 스켈레톤 코드들을 많이 이용하는데 이 함수들에서 `disk_sector_t` 타입으로 `sector_number`를 자주 받고 있는 것을 봤다
>- 즉, `anon_page`가 담아야 할 정보는 이게 맞는 듯?

---
#### 2.5.3.  완성본 
```c
bool anon_initializer(struct page *page, enum vm_type type, void *kva) {
  if (VM_TYPE(type) != VM_ANON) {
    return false;
  }

  /* Set up the handler */
  page->operations = &anon_ops;
  struct anon_page *anon_page = &page->anon;       // 요놈
  memset(anon_page, 0, sizeof(struct anon_page));  // 메타데이터를 초기화
  memset(kva, 0, PGSIZE);  // 익명 페이지랑 매핑된 실제 물리값들 초기화
  // anon_page->slot_idx = SIZE_MAX; 이건 나중에 swap에서 필요
  return true;
}
```
- 익명 페이지는 파일에서 내용을 읽어오지 않기 때문에 초기 상태에서는 0으로 채워진 빈 페이지를 만든다.
- 이를 위해 `memset`을 활용하여 **깨끗한 메모리 영역을 보장**

---
### 2.6.  관련 메서드 - file_backed_initializer
이건 (`mmap` 구현하면서 구현)



---
## 3.  setup stack

>[!QUESTION] 기존 Load 방식 vs VM 기준 Load 방식
>1. 기존 Load
>	- single page 할당 및 세팅
>	- 스택 setup
>2. VM 기준 Load 추가된 점 
>	- vm_entry(Pintos에서는 page)를 생성
>	- initialization 등록
>	- hash table에 vm_entry 삽입 
>	- stack이 성장 가능 

![Pasted image 20250930110514.png](/img/user/supporter/image/Pasted%20image%2020250930110514.png)

