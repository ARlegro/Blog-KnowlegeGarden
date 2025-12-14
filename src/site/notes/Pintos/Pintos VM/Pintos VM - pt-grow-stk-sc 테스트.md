---
{"dg-publish":true,"permalink":"/Pintos/Pintos VM/Pintos VM - pt-grow-stk-sc 테스트/","noteIcon":"","created":"2025-12-03T14:52:52.680+09:00","updated":"2025-12-13T18:25:31.692+09:00"}
---


### 목차

- [[#1.  pt-grow-stk-sc|1.  pt-grow-stk-sc]]
	- [[#1.  pt-grow-stk-sc#1.1.  테스트 코드|1.1.  테스트 코드]]
	- [[#1.  pt-grow-stk-sc#1.2.  문제|1.2.  문제]]
	- [[#1.  pt-grow-stk-sc#1.3.  분석 및 해결|1.3.  분석 및 해결]]
	- [[#1.  pt-grow-stk-sc#1.4.  일부 해결 후 추가 오류|1.4.  일부 해결 후 추가 오류]]
		- [[#1.4.  일부 해결 후 추가 오류#1.4.1.  분석|1.4.1.  분석]]
		- [[#1.4.  일부 해결 후 추가 오류#1.4.2.  ✅해결|1.4.2.  ✅해결]]
	- [[#1.  pt-grow-stk-sc#1.5.  추가 문제 발생 💢|1.5.  추가 문제 발생 💢]]
	- [[#1.  pt-grow-stk-sc#1.6.  다른 해결들|1.6.  다른 해결들]]
	- [[#1.  pt-grow-stk-sc#1.7.  추가 문제|1.7.  추가 문제]]
	- [[#1.  pt-grow-stk-sc#1.8.  최종 해결 회고|1.8.  최종 해결 회고]]

---
## 1.  pt-grow-stk-sc

---
### 1.1.  테스트 코드 
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
### 1.2.  문제 
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
### 1.3.  분석 및 해결 
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
분명 syscall_read는 정상적으로 읽어야 한다.
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
어디서 문제가 발생한거지? 디버깅하는데 첫 if문에서 발생했다.
첫 if문에 조건이 3개나 있어서 뭐가 문젠지 알아보기 위해 하나하나 분리해봤다.
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
`pml4_get_page` 여기가 문젠데?
뭐지? 이전 project 1,2까지는 잘 됐는데... 일단 지워볼까? 하고 지워봤다.

그 결과 PF가 나야되는 주소를 가진 buffer가 file_read 함수 인자로 넘어가고, `page_fault`를 일으키는 것을 볼 수 있었다.
![Pasted image 20251004142414.png](/img/user/supporter/image/Pasted%20image%2020251004142414.png)

---
### 1.4.  일부 해결 후 추가 오류
>이전 해결 : `check_page` 로직 검증을 느슨하게 해서 원하는 곳에서 PF가 나게 했다.

기대한대로 `Page_fault`가 정상적인 구간에서 발생했지만 아래와 같은 오류가 발생했다.
```c
FAIL tests/vm/pt-grow-stk-sc
Kernel panic in run: PANIC at ../../userprog/exception.c:95 in kill(): Kernel bug - unexpected interrupt in kernel
Call stack: 0x80042197f9 0x800421ef65 0x800421f11c 0x80042098ea 0x8004209d42 0x80042157d9 0x8004214bbe 0x80042228fa 0x80042217de 0x800421fca4 0x800421f437 0x800421f193 0x4001fd 0x400dc0
Translation of call stack:
0x00000080042197f9: debug_panic (lib/kernel/debug.c:32)
0x000000800421ef65: kill (userprog/exception.c:101)
0x000000800421f11c: page_fault (userprog/exception.c:151)
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

#### 1.4.1.  분석 
로그만 보면 PF가 나고 제대로 해결이 안된채로 Kill까지 간것 같다.
이 테스트 목적을 돌이켜보자
- system-call내부에서 stack에 접근해도 적절하게 stack이 growth되도록 하는 것 


그렇다면 stack성장을 담당하는 아래 코드를 주의 깊게 다시 봐야할 것 같다.
내가 빼먹은게 뭘까???
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
내가 빼먹은게 뭘까???에 대해 생각하다가 인자에서 힌트를 얻었다.
- 인자에서 `vm_try_handle_fault`함수 인자에는 user라는 값이 있는데 나는 여태껏 이 인자를 사용하지 않고 `stack_growth`, `vm_try_handle_fault`를 구현했다.
- 그렇다면 '이걸 이용해야 완벽한 코드가 되지 않을까?' 라는 생각이 들었다.

>[!tip] `vm_try_handle_fault`에 넘어온 user의 의미
>유저 모드에서 발생한건지 아닌지⭐

현재 내가 노리는 테스트코드에서의 Page Fault는 System Call 내부의 커널모드에서 발생하는 것이다.
그리고 디버깅 결과 아래 사진 화살표처럼 바로 `can_grow_stack` 로직으로 가는 것을 볼 수 있었다. 또한 `if (falut_addr < rsp - 32) return false`에서 false를 반환하고 종료됐다.
![Pasted image 20251004143820.png](/img/user/supporter/image/Pasted%20image%2020251004143820.png)
![Pasted image 20251004143918.png](/img/user/supporter/image/Pasted%20image%2020251004143918.png)

흠... 뭘까 기존 다른 테스트는 통과됐는데 가까운지에 대한 테스트를 제외해야하나? GitBook을 다시보자 

다시봐도 뭐가 없는데??
커널 모드랑 유저모드 뭐가 다르게 처리해야되는지 GPT한테 물어보니 명확한 답이 나왔다.
- 현재 can_grow_stack에 넘어온 레지스터, 스택 값들은 커널 모드일 경우 커널 기준이다.
- 즉, 스택이 유저 스택이 아니라 커널 스택이고 rsp도 유저의 stack_pointer가 아니라 커널의 stack_pointer인 것.
- 따라서 엉뚱한 if조건을 대고 있던 것이다.

#### 1.4.2.  ✅해결 
- rsp를 이용한 검사는 user모드일 경우에만 하도록 한다.
```c
bool can_grow_stack(const struct intr_frame *f, void *addr, bool is_user_mode) {
	...
  if (fault_addr >= stack_top) return false;  // 스택 위 접근
  // ✅ 유저모드일 때만 rsp를 활용한 로직 
  if (is_user_mode) {
    if (fault_addr < rsp - 32) return false;    // 가까운지 먼지 (32byte Cuz 4B)
  }
  ....
  
}
```


---
### 1.5.  추가 문제 발생 💢
> 다른 테스트 파일인 pt-bad-read가 다시 에러뜨기 시작했다.

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

---
### 1.6.  다른 해결들

*분석*
- 이 테스트도 결국 **read 시스템 콜을 통해 잘못된 주소를 처리하는 것**을 테스트하는 것이다.
- 4096이라는 **스택 성장이 불가능한 범위의 주소를 제대로 처리하는지**인데
- 문제는 내가 이전에 `pt-grow-stk-sc`를 해결하기 위해 해놓은 can_grow_stack로직에서 kernel모드일 경우 정상적으로 스택이 성장되도록 처리했다.

*해야하는 일* 
- 커널 모드여도 user의 스택 범위를 넘어가면 스택성장이 안되고 syscall_exit처리를 할 수 있도록 해야했다.

*방법*
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

1. *처리 - 하드코딩일수도?*
```c
  if (vm_try_handle_fault(f, fault_addr, user, write, not_present)) return;
  // ✅ 추가 : 커널 모드에서 vm처리도 안된 것 중 user영역 접근하려는 
	if (!user && is_user_vaddr(fault_addr)) {
    syscall_exit(-1);
    return;
  }
```
- 3번 처리는 하드코딩일 수 있지만 일단은 이렇게 해보자

---
### 1.7.  추가 문제 
```bash
Kernel panic in run: PANIC at ../../threads/synch.c:212 in lock_acquire(): assertion `!lock_held_by_current_thread(lock)' failed.
```
락을 쥔채로 뭐 panic떠서? 그런가본데?
하... syscall_read에서 락 건채로 아래 코드 부분인데 
```c
  lock_acquire(&filesys_lock);  // 파일 시스템 접근 보호를 위해 락 획득
  int bytes_write = file_write(file, buffer, size);  // 파일에 데이터 쓰기 수행
  lock_release(&filesys_lock);                       // 파일 연산 후 락 해제
```
근데 내가 Project 2 User Prog할 때 All-Pass하신 동기분 얘기로는 **`file_write()`내부에서 lock이 있어서 굳이 `lock_acquire()`코드를 하지 않아도 됐다는** 말이 생각났고 제거를 해보았다.
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
### 1.8.  최종 해결 회고 
여러가지 방법으로 해결했다.
1. 주소 검증 완화 : vm버전으로 프로그램 시 `pml4_get_page`체크 제거
2. 커널모드에서도 stack_groth시 올바른 user스택의 rsp를 사용해서 검사하도록 
3. 불필요한 락 제거 


