---
{"dg-publish":true,"permalink":"/Pintos/Pintos VM/Pintos VM - VM 버전의 Copy/","noteIcon":"","created":"2025-12-03T14:52:52.667+09:00","updated":"2025-12-13T18:25:31.670+09:00"}
---

### 0.1.  목차

[[#supplemental_page_table_copy|supplemental_page_table_copy]]
- [[#supplemental_page_table_copy#1.  개념 공부|1.  개념 공부]]
	- [[#1.  개념 공부#1.1.  하는 역할?|1.1.  하는 역할?]]
	- [[#1.  개념 공부#1.2.  복사 전략|1.2.  복사 전략]]
		- [[#1.2.  복사 전략#1.2.1.  Deep Copy|1.2.1.  Deep Copy]]
		- [[#1.2.  복사 전략#1.2.2.  Cow|1.2.2.  Cow]]
	- [[#1.  개념 공부#1.3.  주의점 ⚠|1.3.  주의점 ⚠]]
- [[#supplemental_page_table_copy#2.  단서 모으기|2.  단서 모으기]]
	- [[#2.  단서 모으기#2.1.  GitBook 내용|2.1.  GitBook 내용]]
	- [[#2.  단서 모으기#2.2.  추측|2.2.  추측]]
	- [[#2.  단서 모으기#2.3.  초기 코드|2.3.  초기 코드]]
- [[#supplemental_page_table_copy#3.  최종 구현 코드|3.  최종 구현 코드]]

[[#테스트|테스트]]
- [[#테스트#1.  fork-once|1.  fork-once]]
	- [[#1.  fork-once#1.1.  테스트 코드 및 에러|1.1.  테스트 코드 및 에러]]
	- [[#1.  fork-once#1.2.  의문|1.2.  의문]]
	- [[#1.  fork-once#1.3.  디버그 결과|1.3.  디버그 결과]]
	- [[#1.  fork-once#1.4.  추가 문제|1.4.  추가 문제]]
	- [[#1.  fork-once#1.5.  디버그 결과 및 해결|1.5.  디버그 결과 및 해결]]
- [[#테스트#2.  그 외 테스트|2.  그 외 테스트]]

---

>[!example] 임시 구현되어 있던 코드 살펴보기 
Fork, Copy 로직 외 이전 테스트들을 위해 컴파일 되도록 임시 구현했던 코드 살펴보기 
```c
bool supplemental_page_table_copy(struct supplemental_page_table *dst UNUSED,
                                  struct supplemental_page_table *src UNUSED) {
  struct hash_iterator i;
  hash_first(&i, src);
  while (hash_next(&i)) {
    struct hash_elem *h = hash_cur(&i);
    hash_insert(dst, h);
  }
  return true;
}
```
- 이렇게 해서 컴파일만 되도록 했다.
- 하지만 `fork` 관련 테스트는 전부 실패한다.
- `fork` 프로세스 중 `VM`관련 로직은 이 코드가 거의 전부이기에 이 코드만 깊게 공부하면 될 것이다.

근데 GitBook에는 자세히 나와있지 않아서 따로 공부해야 했다. 

---
# supplemental_page_table_copy

---
## 1.  개념 공부 
참고 : SPT란? ➡ [[Pintos/Pintos VM/Pintos VM - 구조체 및 관련 개념 정리\|Pintos VM - 구조체 및 관련 개념 정리]] - Page Table vs SPT 

---
### 1.1.  하는 역할? 
> 부모 프로세스의 SPT(보조 페이지 테이블)를 자식용 SPT로 동일하기 재구성 

부모가 가진 각 페이지마다의 정보들(ex. 타입, 접근권한, VA, lazy_load 등)을 자식용 페이지에 담아서 자식용 SPT에 삽입한다.

---
### 1.2.  복사 전략 

복사 전략은 2가지가 있다.
- Deep Copy
- Cow

#### 1.2.1.  Deep Copy
> 메타데이터들을 전부 깊은 복사 

---
#### 1.2.2.  Cow 
> 부모/자식의 PTE를 일시적으로 read-only로 공유하고 write fault나면 복사 (지연 복사)

---
### 1.3.  주의점 ⚠

1. *얕은 복제 금지*
	- 단순히 부모의 `hash_elem`나 `page` 타입을 그대로 SPT에 `Insert`하지 않는다.
	- 모든 메타데이터도 깊은 복제가 원칙 
	  
2. *타입별 처리 다르게 하기*
	- `UNINIT`
	- 익명 페이지 
	- 파일 기반 페이지 : file 객체 독립적 참조하도록 
	- `STACK` : ANON + 스택 마커 유지 
	  
3. *핀 처리* 
	- Deep Copy 중에는 evict 방지용 pin을 걸고 복사.
	- 끝나면 해제 ㄱ 

---
## 2.  단서 모으기 
---
### 2.1.  GitBook 내용 
> src부터 dst까지 supplemental page table를 복사하세요. 

```c
bool supplemental_page_table_copy (struct supplemental_page_table *dst, struct supplemental_page_table *src);
```

- 이것은 자식이 부모의 실행 context를 상속할 필요가 있을 때 사용됩니다.(ex. `fork()`). 
- `src`의 `supplemental page table`를 **반복하면서 `dst`의 `supplemental page table`의 엔트리의 정확한 복사본을 만드세요.** 
- 당신은 초기화되지않은(`uninit`) 페이지를 할당하고 그것들을 바로 요청할 필요가 있을 것입니다.
	- You will need to allocate uninit page and claim them immediately.


---
### 2.2.  추측 
일단은 hash 자료구조로 된 table을 순회하면서 copy할 필요가 있다.
그렇다면 당연히 `hash.c` 파일을 통해 어떤 스켈레톤 코드가 준비되어 있나 보면 된다.

우선 순회인 iterate관련 스켈레톤 코드는 아래와 같고 애네를 잘 활용하면 될 것 같다.
```c
/* 해시 테이블 반복자(iterator). */
struct hash_iterator {
  struct hash *hash;      /* 현재 순회 중인 해시 테이블. */
  struct list *bucket;    /* 현재 가리키는 버킷. */
  struct hash_elem *elem; /* 현재 버킷 내의 해시 요소. */
};

/* Iteration. */
void hash_apply(struct hash *, hash_action_func *);
void hash_first(struct hash_iterator *, struct hash *);
struct hash_elem *hash_next(struct hash_iterator *);
struct hash_elem *hash_cur(struct hash_iterator *);
```


`hash_first`코드를 보면 `iterator`를 초기화하는 로직 같아 보이니까 무조건 써야될 듯?
```c
/* Initializes I for iterating hash table H.
   Iteration idiom:
   
	   struct hash_iterator i;
	   hash_first (&i, h);
	   while (hash_next (&i))
	   {
		   struct foo *f = hash_entry (hash_cur (&i), struct foo, elem);
		   ...do something with f...
	   }

   Modifying hash table H during iteration, using any of the
   functions hash_clear(), hash_destroy(), hash_insert(),
   hash_replace(), or hash_delete(), invalidates all
   iterators. */
void hash_first(struct hash_iterator *i, struct hash *h) {
  i->hash = h;
  i->bucket = i->hash->buckets;
  i->elem = list_elem_to_hash_elem(list_head(i->bucket));
}
```

이걸 어떻게 활용할지 생각하는데 주석에 자세한 설명이 있네???

---
### 2.3.  초기 코드
```c
bool supplemental_page_table_copy(struct supplemental_page_table *dst UNUSED,
                                  struct supplemental_page_table *src UNUSED) {
  struct hash_iterator i;
  hash_first(&i, src);

  while (hash_next(&i)) {
    struct hash_elem *h = hash_cur(&i);
    hash_insert(dst, h);
  }
}
```
일단 임시로 구현해놨다.
어차피 나중에 바꿀거라서 

---
## 3.  최종 구현 코드 
> 페이지 타입별, Load 여부별로 Copy를 다르게 해야 한다.

| 상황                  | 부모 페이지 상태  | 자식 처리 방식                 |
| ------------------- | ---------- | ------------------------ |
| UNINIT              | 아직 lazy 상태 | `frame = NULL`, aux 복사   |
| ANON / FILE (로드됨)   | 이미 메모리에 있음 | `frame` 새로 할당 후 `memcpy` |
| ANON / FILE (로드 안됨) | lazy 로딩 예정 | 역시 `frame = NULL` 유지     |

![Pasted image 20251015143532.png](/img/user/supporter/image/Pasted%20image%2020251015143532.png)
사진과 같은 그림으로 함수를 구현했다.
(`spt_copy` = `supplemental_page_table_copy`)

```C
static bool copy_uninit_page(struct supplemental_page_table* dst,
                             struct page* src) {
  struct lazy_load_aux* copied_aux = NULL;
  if (src->uninit.aux != NULL) {
    copied_aux = copy_aux(src->uninit.aux);
  }

  if (!vm_alloc_page_with_initializer(src->uninit.type, src->va, src->writable,
                                      src->uninit.init, copied_aux)) {
    file_close(copied_aux->file);
    free(copied_aux);
    return false;
  }

  return true;
}

/**
 * file-backed는 즉시로딩이긴한데 주의할 점 : reopen을 해야한다는 것
 * file.c의 initializer에서 reopen을 할까 고민을 했지만 그렇게 되면 처음 open도
 * reopen해야해서 이상함 그래서 어차피 올라온 memory는 memcpy쓰고 file포인터
 * 가진 부분만 교체하면 되니까 이런 방식 사용
 */
static bool copy_loaded_page(struct supplemental_page_table* dst,
                             struct page* src) {
  enum vm_type type = VM_TYPE(src->operations->type);
  void* va = src->va;
  struct page* dst_page = NULL;
  void* kva = NULL;

  if (!vm_alloc_page_with_initializer(type, va, src->writable, NULL, NULL)) {
    return false;
  }

  if (!vm_claim_page(va)) return false;
  dst_page = spt_find_page(dst, src->va);
  if (dst_page == NULL) return false;

  if (type == VM_FILE) {
    // 나중에 모듈화하던가
    struct file_page* dst_fp = &dst_page->file;
    struct file_page* src_fp = &src->file;
    dst_fp->file = file_reopen(src_fp->file);
    dst_fp->ofs = src_fp->ofs;
    dst_fp->page_read_bytes = src_fp->page_read_bytes;
    dst_fp->page_zero_bytes = src_fp->page_zero_bytes;
    dst_fp->is_writable = src_fp->is_writable;
  }

  kva = dst_page->frame->kva;
  memcpy(kva, src->frame->kva, PGSIZE);
  return true;
}

// 분기 처리 담당하는 함수 (uninit vs loaded)
static bool copy_page(struct supplemental_page_table* dst, struct page* src) {
  bool copy_succ = false;
  switch (VM_TYPE(src->operations->type)) {
    case VM_UNINIT:
      copy_succ = copy_uninit_page(dst, src);
      break;

    default:
      copy_succ = copy_loaded_page(dst, src);
      break;
  }
  return copy_succ;
}

/* Copy supplemental page table from src to dst */
// 부모의 SPT 엔트리를 순회하면서 lazy load 정보까지 복사한다. */
bool supplemental_page_table_copy(struct supplemental_page_table* dst UNUSED,
                                  struct supplemental_page_table* src UNUSED) {
  struct hash_iterator i;
  hash_first(&i, &src->page_table);

  while (hash_next(&i)) {
    struct page* page = hash_entry(hash_cur(&i), struct page, hash_elem);
    if (page == NULL) return false;

    if (!copy_page(dst, page)) {
      hash_clear(&dst->page_table, destruct_hash_elem);
      return false;
    }
  }

  return true;
```


---

# 테스트 
---
## 1.  fork-once

> 목적 : Project 3인 VM에서도 fork가 제대로 작동되는지 테스트하는 코드 

---
### 1.1.  테스트 코드 및 에러 
```c
void test_main(void) {
  int pid;
  if ((pid = fork("child"))) {
    int status = wait(pid);
    msg("Parent: child exit status is %d", status);
  } else {
    msg("child run");
    exit(81);
  }
}
```

*💢에러*
```bash
Boot complete.
Putting 'fork-once' into the file system...
Executing 'fork-once':
(fork-once) begin
child: exit(-1)
```

---
### 1.2.  의문 
계속 이상한 곳에서 에러가 뜬다. (부모 페이지 복사하는 곳 내부에서)
부모 페이지를 복사할 때 loaded된 page를 copy하는 로직에서 계속 에러가 떴다.
```c 
static bool copy_loaded_page(struct supplemental_page_table *dst,

                             struct page *src) {

  enum vm_type type = page_get_type(src);
  struct page *dst_page = NULL;
	void *kva = NULL;
  ...
  
  dst_page = spt_find_page(dst, src->va);  
  ...
    kva = dst_page->frame->kva;
    // ❌ 여기서 그냥 child 종료
    memcpy(kva, src->frame->kva, PGSIZE);
  }

  return true;
```

---
### 1.3.  디버그 결과 
부모의 페이지(`src`)를 기반으로 copy하려하는데 스켈레톤 코드인 `page_get_type`을 통해 얻은 `type`은 `VM_ANON`인데 디버그 창의 `src->operations->type` 은 `VM_UNIT`이었다.
![Pasted image 20251005135836.png](/img/user/supporter/image/Pasted%20image%2020251005135836.png)

*❓왜 이런 결과가 나왔을까❓*
- 스켈레톤 코드인 `page_get_type`을 자세히 보면 이 메크로는 현재 타입이 아니라 최종 변환 타입을 반환해주고 있었다.
- 즉, `page_get_type` 메크로는 현재 페이지 상태를 보기 위한 적절한 메크로가 아니었던 것이다.

```c
enum vm_type page_get_type(struct page *page) {
  int ty = VM_TYPE(page->operations->type);
  switch (ty) {
	  // UNINIT일 경우 최종 타입으로 반환
    case VM_UNINIT:
      return VM_TYPE(page->uninit.type);
    default:
      return ty;
  }
```

---
### 1.4.  추가 문제 
이번엔 대기가 길지 않고 바로 출력값으로 뜨기는 하는데 기대값은 아니다
```bash
Acceptable output:
  (fork-once) begin
  (fork-once) child run
  child: exit(81)
  (fork-once) Parent: child exit status is 81
  (fork-once) end
  fork-once: exit(0)
Differences in `diff -u' format:
  (fork-once) begin
- (fork-once) child run
- child: exit(81)
- (fork-once) Parent: child exit status is 81
+ (fork-once) Parent: child exit status is -1
  (fork-once) end
  fork-once: exit(0)
FAIL
test 1/1 finish
```

---
### 1.5.  디버그 결과 및 해결 
디버그 결과
- fork중 부모의 `uninit_page`를 복사하는 과정에서 err가 떴다.
```c
static bool copy_uninit_page(struct supplemental_page_table *dst,
                             struct page *src) {
  void *va = src->va;
  struct page *dst_page = NULL;
  void *kva = NULL;
  struct lazy_load_aux *copied_aux = copy_aux(src->uninit.aux);
  if (copied_aux == NULL) return false;


  if (!vm_alloc_page_with_initializer(VM_UNINIT, va, src->writable,
                                     src->uninit.init, copied_aux)) {
		// ❌ 여기오면 안되는데 열로 옴                                      
    goto err;
  }
  return true;

err:
  file_close(copied_aux->file);
  free(copied_aux);
  return false;
}
```
uninit_page 복사 로직 중 `vm_alloc_page_with_initializer` 호출이 실패했다.
왜일까? 더 자세하게 살펴보던 중 `vm_alloc_page_with_initializer` 내부 switch문 속 비정상 분기로 흘러가는게 보여졌다.
```c
bool vm_alloc_page_with_initializer(enum vm_type type, void *upage, bool writable, vm_initializer *init, void *aux) {
		....
    bool (*initializer)(struct page *, enum vm_type, void *kva) = NULL;
    switch (VM_TYPE(type)) {

      case VM_ANON:
        break;

		  /...

      default:
	      // ❌ 오면 안되는데 오게 됨 
        free(page);
        goto err;
    }
```
*❓오류 이유* 
- `vm_alloc_page_with_initializer`의 첫 번째 인자로 최종 page형태가 아니라 현재 page형태를 넘겨줬던 것이었다.
- 이 함수는 최종 page형태로 예약을 거는 함수라서 현재 page 형태로 인자를 넘기면 안된다.

*✅해결 및 결과* 
```c
// ❌
if (!vm_alloc_page_with_initializer(VM_UNINIT, va, src->writable, src->uninit.init, copied_aux)) {
    goto err;
}

//✅
if (!vm_alloc_page_with_initializer(src->uninit.type, va, src->writable, src->uninit.init, copied_aux)) {
    goto err;
}
```


---
## 2.  그 외 테스트
그 외 테스트는 Copy로직들을 조정하다보니 자연스럽게 해결됐다. 2개 빼고
2개에 대한 테스트는 다른 페이지 링크에 남겨뒀으니 참고 바란다.
(링크 : [[Pintos/Pintos VM/Pintos VM - 마지막 테스트 fork-read\|Pintos VM - 마지막 테스트 fork-read]])