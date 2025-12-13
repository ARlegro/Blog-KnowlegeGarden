---
{"dg-publish":true,"permalink":"/Pintos/Pintos VM/Pintos VM - Stack Growth/","noteIcon":"","created":"2025-12-03T14:52:52.710+09:00","updated":"2025-12-13T09:26:33.323+09:00"}
---



### 0.1.  목차

- [[#1.  GitBook 내용 및 추측|1.  GitBook 내용 및 추측]]
- [[#2.  스택 확장 조건|2.  스택 확장 조건]]
- [[#3.  setup_stack|3.  setup_stack]]
	- [[#3.  setup_stack#3.1.  참고 내용 및 추측|3.1.  참고 내용 및 추측]]
	- [[#3.  setup_stack#3.2.  기존 setup_stack|3.2.  기존 setup_stack]]
	- [[#3.  setup_stack#3.3.  트러블|3.3.  트러블]]

[[#테스트 부수기|테스트 부수기]]
- [[#테스트 부수기#1.  pt-big-stk-obj|1.  pt-big-stk-obj]]
	- [[#1.  pt-big-stk-obj#1.1.  테스트 코드 및 오류|1.1.  테스트 코드 및 오류]]
	- [[#1.  pt-big-stk-obj#1.2.  문제점 💢|1.2.  문제점 💢]]
		- [[#1.2.  문제점 💢#1.2.1.  ✅문제 지점 확인|1.2.1.  ✅문제 지점 확인]]
		- [[#1.2.  문제점 💢#1.2.2.  추측 및 해결|1.2.2.  추측 및 해결]]
- [[#테스트 부수기#2.  pt-write-code2|2.  pt-write-code2]]
	- [[#2.  pt-write-code2#2.1.  테스트 코드|2.1.  테스트 코드]]
	- [[#2.  pt-write-code2#2.2.  문제 💢|2.2.  문제 💢]]
	- [[#2.  pt-write-code2#2.3.  추측 및 해결|2.3.  추측 및 해결]]
- [[#테스트 부수기#3.  pt-grow-stk-sc, pt-bad-read|3.  pt-grow-stk-sc, pt-bad-read]]
	- [[#3.  pt-grow-stk-sc, pt-bad-read#3.1.  테스트 코드|3.1.  테스트 코드]]
	- [[#3.  pt-grow-stk-sc, pt-bad-read#3.2.  문제|3.2.  문제]]
	- [[#3.  pt-grow-stk-sc, pt-bad-read#3.3.  분석 및 해결|3.3.  분석 및 해결]]
	- [[#3.  pt-grow-stk-sc, pt-bad-read#3.4.  일부 해결 후 추가 오류|3.4.  일부 해결 후 추가 오류]]
		- [[#3.4.  일부 해결 후 추가 오류#3.4.1.  분석|3.4.1.  분석]]
		- [[#3.4.  일부 해결 후 추가 오류#3.4.2.  ✅해결 (사실 미완성 해결임)|3.4.2.  ✅해결 (사실 미완성 해결임)]]
	- [[#3.  pt-grow-stk-sc, pt-bad-read#3.5.  추가 문제 발생 💢|3.5.  추가 문제 발생 💢]]
		- [[#3.5.  추가 문제 발생 💢#3.5.1.  분석|3.5.1.  분석]]
		- [[#3.5.  추가 문제 발생 💢#3.5.2.  해야하는 일|3.5.2.  해야하는 일]]
		- [[#3.5.  추가 문제 발생 💢#3.5.3.  해결|3.5.3.  해결]]
	- [[#3.  pt-grow-stk-sc, pt-bad-read#3.6.  추가 문제|3.6.  추가 문제]]
	- [[#3.  pt-grow-stk-sc, pt-bad-read#3.7.  최종 해결|3.7.  최종 해결]]




## 1.  GitBook 내용 및 추측 

프로젝트 2에서 스택은 `USER_STACK`에서 시작하는 단일 페이지였으며, 프로그램은 이 크기(4KB)로 제한하여 실행했습니다. 이제 **스택이 현재 크기를 초과하면 필요에 따라 추가 페이지를 할당**합니다

**추가 페이지는 스택에 접근하는 경우에만 할당**합니다. 스택에 접근하는 경우와 아닌 경우를 구별하는 휴리스틱을 고안하세요. ⭐ ⬅ 이게 핵심이네 

User program은 스택 포인터 아래의 스택에 쓸 경우 버그가 발생하는데, 이는 일반적인 실제 OS가 스택의 데이터를 수정하는 시그널을 전달하기 위해 프로세스를 언제든지 중단할 수 있기 때문입니다. 하지만 x86-64 PUSH 명령어는 **스택 포인터를 조정하기 전에 접근 권한을 검사**하므로, **스택 포인터 아래 8바이트에 대해서 `Page Fault`를 발생**시킬 수 있습니다.

당신은 User Program의 스택 포인터의 현재 값을 얻을 수 있어야 합니다. System Call 또는 User Program에 의해 발생한 Page Fault 루틴에서 각각 `syscall_handler()` 또는 `page_fault()`에 전달된 `struct intr_frame`의 `rsp`멤버에서 검색할 수 있습니다. 

잘못된 메모리 접근을 감지하기 위해 `Page Fault`에 의존하는 경우, 커널에서 Page Fault가 발생하는 경우도 처리해야 합니다. 프로세스가 스택 포인터를 저장하는 것은 예외로 인해 유저 모드에서 커널 모드로 전환될 때 뿐이므로 `page_fault()`로 전달된 `struct intr_frame`에서 `rsp`를 읽으면 유저 스택 포인터가 아닌 정의되지 않은 값을 얻을 수 있습니다. 유저 모드에서 커널 모드로 전환 시 `rsp`를 `struct thread`에 저장하는 것과 같은 다른 방법을 준비해야 합니다.

> 스택 증가 기능을 구현합니다.

이 기능을 구현하려면 먼저 `vm/vm.c`에 있는 `vm_try_handle_fault`를 수정하여 스택 증가를 확인합니다. 스택 증가를 확인한 후에는 `vm/vm.c`에서 `vm_stack_growth`를 호출하여 스택을 증가시켜야 합니다. `vm_stack_growth`를 구현하세요.

```c
bool vm_try_handle_fault (struct intr_frame *f, void *addr, bool user, bool write, bool not_present);
```
이 함수는 `Page Fault` 예외를 처리하는 동안 `userprog/exception.c`에서 `page_fault()`로 호출됩니다. 이 함수에서는 `Page Fault`가 스택을 증가시켜야하는 경우에 해당하는지 아닌지를 확인해야 합니다. 스택 증가로 `Page Fault` 예외를 처리할 수 있는지 확인한 경우, `Page Fault`가 발생한 주소로 `vm_stack_growth`를 호출합니다.

```c
void vm_stack_growth (void *addr);
```
**하나 이상의 anonymous 페이지를 할당하여 스택 크기를 늘립니다.** 이로써 `addr`은 `faulted` 주소(폴트가 발생하는 주소) 에서 유효한 주소가 됩니다. 페이지를 할당할 때는 주소를 `PGSIZE`기준으로 내림하세요.

대부분의 OS에서 스택 크기는 절대적으로 제한되어있습니다. 일부 OS는 사용자가 크기 제한을 조정할 수 있게 합니다(예를 들자면, 많은 Unix 시스템에서 ulimit커맨드로 조정할 수 있습니다). 많은 GNU/Linux 시스템에서 기본 제한은 8MB입니다. 이 프로젝트의 경우 스택 크기를 최대 1MB로 제한해야 합니다

이제 모든 stack-growth test case를 통과해야 합니다.


---
## 2.  스택 확장 조건 

*스택 확장 조건* 
1. *not-preset PF어야 함* 
2. *가까운지 체크* 
	- `Fault Address`가 현재 `rsp`보다 약간 아래여야 함 
	- 비교 시 타입은 `unitptr_t` 사용 권장
		- 주소를 담을 수 있는 부호 없는 정수라 주소-정수 연산에 가장 안전 
		- 사용 예시 - 
	  
3. *최종 스택 사이즈 제한*
	- 스택이 너무 커지지 않게 제한두는 것 
	- `USER_STACK - pg_round_down(fault_addr) <= MAX_STACK_SIZE`
	- `MAX_STACK_SIZE`는 자유
		- 1MB 


---
## 3.  setup_stack  

### 3.1.  참고 내용 및 추측 
*참고 : GitBook 내용* 
```text
당신은 스택 할당 부분이  새로운 메모리 관리 시스템에 적합할 수 있도록 userprog/process.c에 있는 setup_stack 을 수정해야 합니다. 
첫 스택 페이지는 지연적으로 할당될 필요가 없습니다. ✅
당신은 페이지 폴트가 발생하는 것을 기다릴 필요 없이 그것(스택 페이지)을 load time 때 커맨드 라인의 인자들과 함께 할당하고 초기화 할 수 있습니다. 
당신은 스택을 확인하는 방법을 제공해야 합니다. 
당신은 vm/vm.h의 vm_type에 있는 보조 marker(예 - VM_MARKER_0)들을 페이지를 마킹하는데 사용할 수 있습니다
```

키워드
- 첫 스택 페이지는 바로 할당
- 커맨드 라인과 할당 **초기화**
- marker?

*초기 `setup_stack`*
```c
/* Create a PAGE of stack at the USER_STACK. Return true on success. */
static bool setup_stack(struct intr_frame *if_) {
  bool success = false;
  void *stack_bottom = (void *)(((uint8_t *)USER_STACK) - PGSIZE);

  /* TODO: Map the stack on stack_bottom and claim the page immediately.
   * TODO: If success, set the rsp accordingly.
   * TODO: You should mark the page is stack. */
  /* TODO: Your code goes here */

  return success;
}
```
- page가 stack이라는 것을 mark하라고?
- makr를 하는 타입이 `vm_type`이라는데 이거는 개별 page 구조체(ex. anon, uninit, file-backed)에 있어야 할 듯?(초기 anon에 있는걸 보니)

근데 일단 인자로 뭐가 들어오는지 알아야 힌트를 얻을 수 있을 것 같다.

---
### 3.2.  기존 setup_stack
초기 stack생성은 지연 로딩과 상관없어서 VM이전의 setup_stack 로직과 크게 다르지 않을 거라고 생각한다. 따라서 기존 로직을 복습 
```c
/* USER_STACK 주소에 0으로 초기화된(page가 모두 0인) 메모리 페이지를 매핑해서
   가장 기본적인 스택 공간을 마련해준다. Create a minimal stack by mapping a
   zeroed page at the USER_STACK */
static bool setup_stack(struct intr_frame *if_) {
  uint8_t *kpage;
  bool success = false;
  
  kpage = palloc_get_page(PAL_USER | PAL_ZERO);
  if (kpage != NULL) {
    success = install_page(((uint8_t *)USER_STACK) - PGSIZE, kpage, true);
    if (success)
      if_->rsp = USER_STACK;
    else
      palloc_free_page(kpage);
  }
  return success;
}
```
```c
/* Adds a mapping from user virtual address UPAGE to kernel
 * virtual address KPAGE to the page table.
 * If WRITABLE is true, the user process may modify the page;
 * otherwise, it is read-only.
 * UPAGE must not already be mapped.
 * KPAGE should probably be a page obtained from the user pool
 * with palloc_get_page().
 * Returns true on success, false if UPAGE is already mapped or
 * if memory allocation fails. */
static bool install_page(void *upage, void *kpage, bool writable) {
  struct thread *t = thread_current();

  /* Verify that there's not already a page at that virtual
   * address, then map our page there. */
  return (pml4_get_page(t->pml4, upage) == NULL &&
          pml4_set_page(t->pml4, upage, kpage, writable));
```

---
### 3.3.  트러블 

초기 구현 : GitBook에서 즉시 로딩을 활용하라 해서 처음에는 `vm_claim_page`를 활용했다.
```c
// ❌ 초기 구현 
static bool setup_stack(struct intr_frame *if_) {
  bool success = false;
  void *stack_bottom = (void *)(((uint8_t *)USER_STACK) - PGSIZE);
  struct thread *cur = thread_current();
  struct page *p = NULL;
  
  success = vm_claim_page(stack_bottom);
  // 흠.. mark를 어따하지????? page 구조체??????
  p = pml4_get_page(cur->pml4, stack_bottom);
  ..
	if (p != NULL) {
		//💢 이건 아닐텐데 ㅡㅡ
		p->operations->type = VM_TYPE(VM_MARKER_0 | VM_ANON);
		p->anon.type = VM_TYPE(VM_MARKER_0 | VM_ANON);
		if_->rsp = stack_bottom;
		success = true;
	}
	return success;
}
```
근데 만들면서 보니 GitBook에 적힌 mark는 어떻게 하지?하면서 별의별 시도를 해봤는데 흠... 만족 스럽지 않았다. 왜냐하면 이 방법이 아닌 것 같은 느낌적인 느낌? (너무 하드코딩 비슷한)
다른 방법이 있을거라 생각하고 mark를 좀 더 우아하게 할 수 있는 방법을 찾아보다가 이전에 lazy_load 관련 로직을 짤 때 `vm_alloc_page_with_initializer`에서 인자로 `TYPE`을 넘겨줬던 것이 생각났다
```C
bool vm_alloc_page_with_initializer(enum vm_type type, void *upage, bool writable, vm_initializer *init, void *aux) {
```
이 함수는 로딩을 예약하는 함수라서 실제 실행은 따로 해주면 되는데 그건 `vm_claim_page`를 활용하면 될 것 같다.


### 3.4.  최종 코드
```c
static bool setup_stack(struct intr_frame* if_) {
  bool success = false;
  struct thread* cur = thread_current();
  struct page* p = NULL;
  void* stack_bottom = (void*)(((uint8_t*)USER_STACK) - PGSIZE);

  // 로드 예약 함수 실행(Marking : VM_MARKER로 스택 페이지인거 마킹)
  if (!vm_alloc_page_with_initializer(VM_MARKER_0 | VM_ANON, stack_bottom, true,
                                      NULL, NULL)) {
    return false;
  }
  
  // page 요청
  if (!vm_claim_page(stack_bottom)) {
    return false;
  }

  // va기준으로 제대로 page구현된거 확인 후 rsp 및 success 조정
  p = pml4_get_page(cur->pml4, stack_bottom);
  if (p != NULL) {
    if_->rsp = USER_STACK;
    success = true;
  }

  return success;
}
```
*동작 흐름*
1. *stack_bottom 계산*
	- stack은 아래로 성장하므로 `- PGSIZE` 해서 거기부터 스택 페이지 생성하도록 활용할 예정 
	  
2. *stack 페이지 Load 예약 - `vm_alloc_page_with_initializer`*
	- `VM_MARKER_0 | ANON` 마킹 추가 : 스택이고 ANON페이지라는 표시 (GitBook에서 이렇게 하라고 하네)
	- Lazy_Load는 하지 않는다 Cuz 바로 Load할 것이기 때문 
	  
3. *실제 Frame 확보 및 매핑 - `vm_claim_page`*
4. *rsp 설정*
	- 유저 모드 진입 직전의 스택 포인터를 설정 



---
# 테스트 부수기 

---
## 1.  pt-big-stk-obj
### 1.1.  테스트 코드 및 오류 
```c
void test_main(void) {
  char stk_obj[65536];
  struct arc4 arc4;

  arc4_init(&arc4, "foobar", 6);
  memset(stk_obj, 0, sizeof stk_obj);
  arc4_crypt(&arc4, stk_obj, sizeof stk_obj);
  msg("cksum: %lu", cksum(stk_obj, sizeof stk_obj));
}
```

처음 결과
```java
Acceptable output:
  (pt-big-stk-obj) begin
  (pt-big-stk-obj) cksum: 3256410166
  (pt-big-stk-obj) end
Differences in `diff -u' format:
  (pt-big-stk-obj) begin
- (pt-big-stk-obj) cksum: 3256410166
- (pt-big-stk-obj) end
+ Page fault at 0x47470000: not present error writing page in user context.

(Process exit codes are excluded for matching purposes.)
=> FAIL
=== pt-big-stk-obj session end ===
test 1/1 finish

=== Test Summary ===
Passed: 0
Failed: 1
```

테스트 목표
- **엄청 큰 배열을 스택에 잡고 스택이 여러 페이지로 연속적으로 성장하는 것을 테스트** 
- 기존 stack_growth테스트와 달리 `char stk_obj[65536];`로 엄청 큼 

---
### 1.2.  문제점 💢
`can_grow_stack` 함수 내 조건 오류

#### 1.2.1.  ✅문제 지점 확인
![Pasted image 20251003163457.png](/img/user/supporter/image/Pasted%20image%2020251003163457.png)

위의 사진처럼 breaking point를 걸어서 어디서 도대체 PF가 나는지 확인했다.
문제는 2번 째 검사문인 `if(fault_addr >= rsp) return false`였다.
이 로직은 fault가 난 지점이 스택 pointer인 rsp보다 높은 위치에 있어야 된다고 생각해서 만든 것이었다.

---
#### 1.2.2.  추측 및 해결 
- 생각해보니 rsp가 당연히 fault_addr보다 낮은건 당연한거아닌가?
- 이 테스트처럼 스택크기를 크게 잡아서 rsp를 확 낮추는 테스트에서는 

큰 로컬 배열을 잡으면 함수 진입 시 rsp가 아래로 크게 내려간다.
그 위에 접근하는 주소들은 대부분 rsp보다 높기 때문에 `fault_addr >= rsp`가 정상 흐름이다.

*잘못알고 있던 점*💢
- 스택 배열을 잡아도 **페이지가 만들어지고나서 rsp가 낮아지는 줄 알았다.**
- 그로 인해 잘못된 검증을 했었다.<br>![Pasted image 20251003165749.png](/img/user/supporter/image/Pasted%20image%2020251003165749.png)
- 허용범위는 저렇다. (근데, `fault_addr`는 보통 rsp보다 크거나 같다)

*✅해결* 
```c
❌
if ((fault_addr - rsp) >= 32) return false;
// 참고 : fault_addr - rsp는 언더플로우로 아주 큰 값이 된다

// ✅ 해결 
if (fault_addr < rsp - 32) return false;   // 32B보다 더 아래면 금지
```
허용 범위에 맞게 stack_growth 검증 로직을 변경했다.


---

## 2.  pt-write-code2
---
### 2.1.  테스트 코드 
```c
/* Try to write to the code segment using a system call.
   The process must be terminated with -1 exit code. */
void test_main(void) {
  int handle;
  
  CHECK((handle = open("sample.txt")) > 1, "open \"sample.txt\"");
  read(handle, (void *)test_main, 1);
  fail("survived reading data into code segment");
}
```
*주석 曰*
- 이 테스트는 read 시스템 콜을 이용해서 code segment에 쓰려고 시도하는 것이다.
- 이러한 시도 시 프로세스는 반드시 종료 코드 -1과 함께 종료해야 한다.

---
### 2.2.  문제 💢
```c
Selected tests: pt-write-code2

Running pt-write-code2 in batch mode... 
$ pintos -m 20 --fs-disk=10 -p tests/vm/pt-write-code2:pt-write-code2 -p ../../tests/vm/sample.txt:sample.txt --swap-disk=4 -- -q -f run 'pt-write-code2'

FAIL tests/vm/pt-write-code2
run: survived reading data into code segment: FAILED
FAIL
test 1/1 finish

=== Test Summary ===
Passed: 0
Failed: 1
```
현재 내 코드 상으로는 code segment 영역에 정상적으로 read가 되어 그 다음 문장인 `fail("survived reading data into code segment");`에 도착했다. 이로 인해 테스트가 fail이 됐다.

---
### 2.3.  추측 및 해결 
테스트 코드를 보면 `read` **시스템 콜 시 오류를 내야 하는 것 같다.**
syscall_read시에 `buffer`로 넘어온 **주소 검증을 좀 더 강화해야** 할 듯?
```C
// ✅기존 valid_poitner 로직 중 check_page 로직 수정
void vali_pointer(const void *user_addr, size_t size) {
	while (byte_left > 0) {
	    if (!check_page(check_ptr)) {
	      syscall_exit(-1);  // 잘못된 주소일 경우, 즉시 프로세스 종료
	    }
	    ....	    
}

.....
bool check_page(const void *user_addr) {
	// 기존 검증 로직 
  if (user_addr == NULL || !is_user_vaddr(user_addr) || pml4_get_page(thread_current()->pml4, user_addr) == NULL){
    return false;
  }
#ifdef VM
		// ✅ 추가로직 
    // VM: spt에서 페이지를 찾아 writable 여부를 확인하기(코드/세그먼트 영역에 쓰기 불가능하도록)
    struct page *p = spt_find_page(&thread_current()->spt, pg_round_down(user_addr));
    if (p != NULL) return p->writable;
#endif
  return true;
}
```
- 기존 `valid_poitner` 로직 중 `check_page` 로직 수정
- 그 주소가 가리키는 페이지가 `writable`이 `false`일 경우 buffer를 쓰지 못 하도록 하는 검증 로직을 추가했다.



---
## 3.  pt-grow-stk-sc, pt-bad-read
> 의도 : **시스템 콜 내부에서 처음으로 해당 스택 주소에 접근**하더라도, **PF가 나고 → 스택이 필요한 경우 자동 확장 → 다시 이어서 진행**

### 3.1.  테스트 코드 
```c
/* This test checks that the stack is properly extended even if
   the first access to a stack location occurs inside a system
   call.

   From Godmar Back. */
void test_main(void) {
  int handle;
  int slen = strlen(sample);
  char buf2[65536];

  /* Write file via write(). */
  CHECK(create("sample.txt", slen), "create \"sample.txt\"");
  CHECK((handle = open("sample.txt")) > 1, "open \"sample.txt\"");
  CHECK(write(handle, sample, slen) == slen, "write \"sample.txt\"");
  close(handle);

  /* Read back via read(). */
  CHECK((handle = open("sample.txt")) > 1, "2nd open \"sample.txt\"");
  CHECK(read(handle, buf2 + 32768, slen) == slen, "read \"sample.txt\"");

  CHECK(!memcmp(sample, buf2 + 32768, slen),
        "compare written data against read data");
  close(handle);
}
```
*주석 曰*
- 시스템 콜 내부에서 stack위치에 첫 접근일지라도 stack이 적절하게 확장되는지 테스트 
---
### 3.2.  문제 
```c
Acceptable output:
  (pt-grow-stk-sc) begin
  (pt-grow-stk-sc) create "sample.txt"
  (pt-grow-stk-sc) open "sample.txt"
  (pt-grow-stk-sc) write "sample.txt"
  (pt-grow-stk-sc) 2nd open "sample.txt"
  (pt-grow-stk-sc) read "sample.txt"
  (pt-grow-stk-sc) compare written data against read data
  (pt-grow-stk-sc) end
Differences in `diff -u' format:
  (pt-grow-stk-sc) begin
  (pt-grow-stk-sc) create "sample.txt"
  (pt-grow-stk-sc) open "sample.txt"
  (pt-grow-stk-sc) write "sample.txt"
  (pt-grow-stk-sc) 2nd open "sample.txt"
  (pt-grow-stk-sc) read "sample.txt"
- (pt-grow-stk-sc) compare written data against read data
- (pt-grow-stk-sc) end
```
테스트 코드에서 read 시스템 콜이 시작되고 그 다음이 제대로 진행되고 있지 않다.


---
### 3.3.  분석 및 해결 
- read부분 근처에서 뭔가 문제가 생긴 것 같아서 read 시스템 콜 위주로 디버깅을 시작했다.
- 그 후 `read` 시스템 콜 로직 중 `vali_pointer` 메서드에서 비정상 종료가 되는 것을 보고 그 부분 로직들을 위주로 breaking_point를 잡아두었다. 
- 이상한 점 발견
	- 분명 정상적인 `user_addr` 주소가 넘어갔는데 주소 값을 복사하는 `const uint8_t *check_ptr = user_addr;`에서 `cannot access memory`라는 표시가 넘어가는 것을 볼 수 있었다.
![Pasted image 20251004134728.png](/img/user/supporter/image/Pasted%20image%2020251004134728.png)
```c
void vali_pointer(const void *user_addr, size_t size) {
  if (size == 0) {
    return;  // 검사할 바이트 수가 0이면 return
  }
  // ❌ 여기서 이상하게 복사되고 
  const uint8_t *check_ptr = user_addr;  // 현재 검사할 위치 (byte 단위 포인터)
  size_t byte_left = size;               // 검사해야 할 남은 바이트 수
 
  while (byte_left > 0) {
    if (!check_page(check_ptr)) {
	    // 💢 read 여기에 도달하게 된다.
      syscall_exit(-1);  
    }  
```
분명 `syscall_read`는 정상적으로 읽어야 한다.
하지만 디버깅 결과 `valid_pointer` -> `!check_page`에서 syscall_exit(-1)이 호출되고 있다.
그 말은 `check_page`가 잘못된건가? 라고 생각할 수 있다.

☑`check_page`로직을 보자 
```c
bool check_page(const void *user_addr) {
  if (user_addr == NULL || !is_user_vaddr(user_addr) || pml4_get_page(thread_current()->pml4, user_addr) == NULL){
    return false;
  }
  
#ifdef VM
    // VM: spt에서 페이지를 찾아 writable 여부를 확인하기(코드/세그먼트 영역에 쓰기 불가능하도록)
    struct page *p = spt_find_page(&thread_current()->spt, pg_round_down(user_addr));
    if (p != NULL) return p->writable;
#endif
  return true;
}
```
- 어디서 문제가 발생한거지? 디버깅하는데 첫 if문에서 발생했다.
- 첫 if문에 조건이 3개나 있어서 뭐가 문젠지 알아보기 위해 하나하나 분리해봤다.

```c
  if (user_addr == NULL){
    return false;
  }
  
  if (!is_user_vaddr(user_addr){
    return false;
  }
	 
	// ❌문제!!! 
	if (pml4_get_page(thread_current()->pml4, user_addr) == NULL){
    return false;
  }  
```
- `pml4_get_page` 여기가 문젠데?
- 뭐지? 이전 project 1,2까지는 잘 됐는데... 일단 지워볼까? 하고 지워봤다.
- 그 결과 PF가 나야되는 주소를 가진 buffer가 file_read 함수 인자로 넘어가고, `page_fault`를 일으키는 것을 볼 수 있었다.<br>![Pasted image 20251004142414.png](/img/user/supporter/image/Pasted%20image%2020251004142414.png)


⭐rVM버전에서 `pml4_get_page` 검증 로직을 제거해야 하는 이유 
- VM을 활용하는 시스템에서는 **아직 매핑되지 않은 주소도 합법**일수도 있다.
- *이전 즉시 로딩에서* : 무조건 주소는 실제 frame과 매핑되어있어야 했다.
- *VM 버전* : Lazy Loading 가능 
  
---
### 3.4.  일부 해결 후 추가 오류
>이전 해결 : `check_page` 로직 검증을 느슨하게 해서 원하는 곳에서 PF(Page Fault)가 나게 했다.

기대한대로 `Page_fault`가 정상적인 구간에서 발생했지만 아래와 같은 오류가 발생했다.
```c
FAIL tests/vm/pt-grow-stk-sc
Kernel panic in run: PANIC at ../../userprog/exception.c:95 in kill(): Kernel bug - unexpected interrupt in kernel
Call stack: 0x80042197f9 0x800421ef65 0x800421f11c 0x80042098ea 0x8004209d42 0x80042157d9 0x8004214bbe 0x80042228fa 0x80042217de 0x800421fca4 0x800421f437 0x800421f193 0x4001fd 0x400dc0
Translation of call stack:
0x00000080042197f9: debug_panic (lib/kernel/debug.c:32)
0x000000800421ef65: kill (userprog/exception.c:101)
0x000000800421f11c: page_fault (userprog/exception.c:151)  // 기대한 곳 
0x00000080042098ea: intr_handler (threads/interrupt.c:331)
0x0000008004209d42: intr_entry (intr-stubs.o:?)
0x00000080042157d9: input_sector (devices/disk.c:422)
0x0000008004214bbe: disk_read (devices/disk.c:220)
0x00000080042228fa: inode_read_at (filesys/inode.c:191)
0x00000080042217de: file_read (filesys/file.c:62)
0x000000800421fca4: syscall_read (userprog/syscall.c:383)
0x000000800421f437: syscall_handler (userprog/syscall.c:131 (discriminator 1))
0x000000800421f193: no_sti (syscall-entry.o:?)
0x00000000004001fd: (unknown)
0x0000000000400dc0: (unknown)
0x0000000000400e10: (unknown)
FAIL
test 1/1 finish
```

#### 3.4.1.  분석 
에러 로그를 보면 `PF`가 나고 제대로 해결이 안 된 채로 Kill까지 간 것 같다.

이 테스트 목적을 돌이켜보자
> 테스트 목적 : `system-call`내부에서 `stack`에 접근해도 **적절하게 `stack`이 `growth`되도록** 하는 것 

그렇다면 stack성장을 담당하는 아래 코드를 주의 깊게 다시 봐야할 것 같다. 내가 빼먹은게 뭘까???
```c

bool vm_try_handle_fault(struct intr_frame *f, void *addr, bool user,
                         bool write, bool not_present) {
  /* addr 없거나 유저영역주소가 아니거나 PTE 존재하는 경우 false*/
  // stack성장 넣으면서 코드 길어질 것 같아서 if문 합침
  if (addr == NULL || !is_user_vaddr(addr) || !not_present) return false;

  struct supplemental_page_table *spt = &thread_current()->spt;
  struct page *page = NULL;
  void *upage = pg_round_down(addr);

  page = spt_find_page(spt, upage);

  if (page) {
    if (!page->writable && write) {
      vm_handle_wp(page);
      return false;  // writable은 false인데 write가 true로 오면 false
    }

    return vm_do_claim_page(page);
  }
  // page가 없으면 stack 확장 여부 판단 필요
  return can_grow_stack(f, addr) ? vm_stack_growth(upage) : false;
}

// 스택 성장 조건 : 스택 bottot에서 어느정도 가깝고 최대스택크기 안 넘어야 됨
bool can_grow_stack(const struct intr_frame *f, void *addr) {
  uintptr_t fault_addr = (uintptr_t)addr;  // fault address
  uintptr_t rsp = (uintptr_t)f->rsp;
  uintptr_t stack_top = (uintptr_t)USER_STACK;

  if (fault_addr >= stack_top) return false;  // 스택 위 접근
  if (fault_addr < rsp - 32) return false;    // 가까운지 먼지 (32byte Cuz 4B)

  uintptr_t stack_size = stack_top - fault_addr;
  if (stack_size > MAX_STACK_SIZE) return false;  // 최대 스택 크기 초과

  return true;
}
```
내가 빼먹은게 뭘까???에 대해 생각하다가 **인자에서 힌트를 얻었다.**
- 인자에서 `vm_try_handle_fault`함수 인자에는 user라는 값이 있는데 나는 여태껏 이 인자를 사용하지 않고 `stack_growth`, `vm_try_handle_fault`를 구현했다.
- 그렇다면 '이걸 이용해야 완벽한 코드가 되지 않을까?' 라는 생각이 들었다.
- 특히 system_call에서의 로직은 커널모드이기 때문에 이를 분기처리 해야하나?라는 생각이 들었다.

>[!tip] `vm_try_handle_fault`에 넘어온 user의 의미
>유저 모드에서 발생한건지 아닌지⭐

*✅활용 시작* 
- 현재 내가 노리는 테스트코드에서의 `Page Fault`는 `System Call` 내부의 커널모드에서 발생하는 것이다.
- 그리고 디버깅 결과 아래 사진 화살표처럼 바로 `can_grow_stack` 로직으로 가는 것을 볼 수 있었다. 또한 `if (falut_addr < rsp - 32) return false`에서 false를 반환하고 종료됐다.<br>![Pasted image 20251004143820.png](/img/user/supporter/image/Pasted%20image%2020251004143820.png)
![Pasted image 20251004143918.png](/img/user/supporter/image/Pasted%20image%2020251004143918.png)

흠... 뭘까 기존 다른 테스트는 통과됐는데 가까운지에 대한 테스트를 제외해야하나? GitBook을 다시보자 
다시봐도 뭐가 없는데??

커널 모드랑 유저모드 뭐가 다르게 처리해야되는지 GPT한테 물어보니 명확한 답이 나왔다.(뒤에 추가 문제들을 보면 사실 이건 명확한 답은 아니었다)
- 현재 `can_grow_stack`에 넘어온 레지스터, 스택 값들은 커널 모드일 경우 커널 기준이다.
- 즉, **인자의 스택이 유저 스택이 아니라 커널 스택**이고 rsp도 유저의 stack_pointer가 아니라 커널의 stack_pointer인 것.
- 따라서 **엉뚱한 if조건**을 대고 있던 것이다.

#### 3.4.2.  ✅해결 (사실 미완성 해결임)
- rsp를 이용한 검사는 user모드일 경우에만 하도록 한다.
```c
bool can_grow_stack(const struct intr_frame *f, void *addr, bool is_user_mode) {
	...
	uintptr_t rsp = (uintptr_t)f->rsp;
	
  if (fault_addr >= stack_top) return false;  // 스택 위 접근
  // ✅ 유저모드일 때만 rsp를 활용한 로직 
  if (is_user_mode) {
    if (fault_addr < rsp - 32) return false;    // 가까운지 먼지 (32byte Cuz 4B)
  }
  ....
}
```
- 일단 유저모드일때만 인자로 넘어온 `intr_frame`내부의 rsp를 활용하기로 했다.


### 3.5.  추가 문제 발생 💢
> 통과를 원했던 `pt-grow-stk-sc.c`는 통과가 됐는데 이전의 통과했단 다른 테스트 파일인 `pt-bad-read`가 다시 에러뜨기 시작했다.

```c
FAIL tests/vm/pt-bad-read
Kernel panic in run: PANIC at ../../userprog/exception.c:96 in kill(): Kernel bug - unexpected interrupt in kernel
Call stack: 0x8004219808 0x800421ef74 0x800421f12b 0x80042098f9 0x8004209d51 0x800422298e 0x80042217f6 0x800421fcd1 0x800421f464 0x800421f1a2 0x40008f 0x400bca 0x400c1a
Translation of call stack:
0x0000008004219808: debug_panic (lib/kernel/debug.c:32)
0x000000800421ef74: kill (userprog/exception.c:102)
0x000000800421f12b: page_fault (userprog/exception.c:157)
0x00000080042098f9: intr_handler (threads/interrupt.c:331)
0x0000008004209d51: intr_entry (intr-stubs.o:?)
0x000000800422298e: inode_read_at (filesys/inode.c:204)
0x00000080042217f6: file_read (filesys/file.c:62)
0x000000800421fcd1: syscall_read (userprog/syscall.c:387)
0x000000800421f464: syscall_handler (userprog/syscall.c:133 (discriminator 1))
0x000000800421f1a2: no_sti (syscall-entry.o:?)
0x000000000040008f: (unknown)
0x0000000000400bca: (unknown)
0x0000000000400c1a: (unknown)
```

```c
void test_main(void) {
  int handle;
  
  CHECK((handle = open("sample.txt")) > 1, "open \"sample.txt\"");
  read(handle, (char *)&handle - 4096, 1);
  fail("survived reading data into bad address");
```


#### 3.5.1.  분석
- 이 테스트도 결국 **read 시스템 콜을 통해 잘못된 주소를 처리하는 것**을 테스트하는 것이다.
- 4096이라는 **스택 성장이 불가능한 범위의 주소를 제대로 처리하는지**인데
- 문제는 내가 이전에 `pt-grow-stk-sc`를 해결하기 위해 해놓은 can_grow_stack로직에서 kernel모드일 경우 정상적으로 스택이 성장되도록 처리했다.

#### 3.5.2.  해야하는 일 
- 커널 모드여도 user의 스택 범위를 넘어가면 스택성장이 안되고 syscall_exit처리를 할 수 있도록 해야했다.

#### 3.5.3.  해결 
1. *스레드 구조체 변경* : 커널 모드에서도 user_stack을 알 수 있도록 thread 구조체 변경 후 init시 
	```c
	// 구조체 추가
	struct thread {
		  uintptr_t user_rsp_snap_shot;
		  
	// USER_STACK으로 초기화
	static void init_thread(struct thread *t, const char *name, int priority) {
		  t->user_rsp_snap_shot = (uint64_t)USER_STACK;
		  
	// syscall 시 주입 
	void syscall_handler(struct intr_frame *f) {		  
	  thread_current()->user_rsp_snap_shot = f->rsp;
	```
2. **커널모드에서 USER_STACK의 rsp확인으로 성장 검증**
	```c
	bool can_grow_stack(const struct intr_frame *f, void *addr, bool is_user_mode) {
		  uintptr_t fault_addr = (uintptr_t)addr;  // fault address
		  uintptr_t rsp = is_user_mode ? (uintptr_t)f->rsp : thread_current()->user_rsp_snap_shot;
		
		  ....
		  if (fault_addr < rsp - 32) return false;    // 가까운지 먼지 (32byte Cuz 4B)
		
		  ...
	return true;
	```

3. *처리 - 하드코딩일수도?*
```c
static void page_fault(struct intr_frame *f) {
  if (vm_try_handle_fault(f, fault_addr, user, write, not_present)) return;
  // ✅ 추가 : 커널 모드에서 vm처리도 안된 것 중 user영역 접근하려는 
	if (!user && is_user_vaddr(fault_addr)) {
    syscall_exit(-1);
    return;
  }
```
- 3번 처리는 하드코딩일 수 있지만 일단은 이렇게 해보자
- 이렇게 처리한 이유
	- 커널이 user가 접근하려는 영역을 대신 접근해주다가 오류가나면 
	- 커널이 문제가 아니라 user가 문제이므로 `syscall_exit`호출로 user process 종료로 마무리 짓는다(user 잘못인데 kill할 필요는 ㄴㄴ)

---
### 3.6.  추가 문제 
```bash
Kernel panic in run: PANIC at ../../threads/synch.c:212 in lock_acquire(): assertion `!lock_held_by_current_thread(lock)' failed.
```
- 락을 쥔채로 뭐 panic떠서? 그런가본데?
- 하... syscall_read에서 락 건채로 아래 코드 부분인데 
```c
	  lock_acquire(&filesys_lock);  // 파일 시스템 접근 보호를 위해 락 획득
	  int bytes_write = file_write(file, buffer, size);  // 파일에 데이터 쓰기 수행
	  lock_release(&filesys_lock);                       // 파일 연산 후 락 해제
```

*✅과거 회상을 통한 해결* 
- 근데 내가 Project 2 User Prog할 때 All-Pass하신 동기분 얘기로는 **`file_write()`내부에서 lock이 있어서 굳이 `lock_acquire()`코드를 하지 않아도 됐다는** 말이 생각났고 제거를 해보았다.
```c
  int bytes_write = file_write(file, buffer, size);  // 파일에 데이터 쓰기 수행
  if (bytes_write < 0) {
    return -1;  // 쓰기가 실패했다면 오류 반환
  }
```

결과 : 전부 통과!!!!!
```text
Passed: 8
  - pt-grow-stack
  - pt-grow-bad
  - pt-big-stk-obj
  - pt-bad-addr
  - pt-bad-read
  - pt-write-code
  - pt-write-code2
  - pt-grow-stk-sc
Failed: 0
```

---
### 3.7.  최종 해결 돌아보기 

여러가지 방법으로 해결했다. 뭐.. 간단하게 핵심 몇 개만 뽑자면 
1. *주소 검증 완화*
	- 지연 로딩방식에서는 `pml4_get_page` 검증 로직 제거 Cuz 유효한 접근일 수도 있음 
	  
2. *올바른 rsp 활용*
	- 커널모드에서도 `stack_groth`시 **올바른 user스택의 rsp를 사용해서 검사**하도록
	  
3. *불필요한 Lock 제거 




