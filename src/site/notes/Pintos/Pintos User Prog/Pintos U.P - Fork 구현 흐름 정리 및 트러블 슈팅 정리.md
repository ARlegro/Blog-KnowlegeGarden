---
{"dg-publish":true,"permalink":"/Pintos/Pintos User Prog/Pintos U.P - Fork 구현 흐름 정리 및 트러블 슈팅 정리/","noteIcon":"","created":"2025-12-03T16:03:22.634+09:00","updated":"2025-12-03T16:09:26.972+09:00"}
---


### 0.1.  목차

- [[#1.  Fork 구현 목적|1.  Fork 구현 목적]]
- [[#2. fork() 주요 복사 대상]]
- [[#3.  사전 확인 포인트|3.  사전 확인 포인트]]
- [[#4.  구현 흐름 공부|4.  구현 흐름 공부]]
	- [[#4.  구현 흐름 공부#4.1.  부모 프로세스가 할 일|4.1.  부모 프로세스가 할 일]]
	- [[#4.  구현 흐름 공부#4.2.  자식 프로세스가 할 일|4.2.  자식 프로세스가 할 일]]
- [[#5.  문제들 모음 💢|5.  문제들 모음 💢]]
	- [[#5.  문제들 모음 💢#5.1.  trouble 1. malloc으로 바구니 만든 점|5.1.  trouble 1. malloc으로 바구니 만든 점]]
	- [[#5.  문제들 모음 💢#5.2.  trouble 2. 부모/자식이 fd_table_entry를 공유해버린 버그|5.2.  trouble 2. 부모/자식이 fd_table_entry를 공유해버린 버그]]
		- [[#5.2.  trouble 2. 부모/자식이 fd_table_entry를 공유해버린 버그#5.2.1.  잘못된 코드|5.2.1.  잘못된 코드]]
		- [[#5.2.  trouble 2. 부모/자식이 fd_table_entry를 공유해버린 버그#5.2.2.  해결 코드 ✅|5.2.2.  해결 코드 ✅]]
	- [[#5.  문제들 모음 💢#5.3.  trouble 3. 가상 페이지 복사 누락|5.3.  trouble 3. 가상 페이지 복사 누락]]
	- [[#5.  문제들 모음 💢#5.4.  trouble 4. 커널 프레임 복사 문제 ⭐|5.4.  trouble 4. 커널 프레임 복사 문제 ⭐]]
		- [[#5.4.  trouble 4. 커널 프레임 복사 문제 ⭐#5.4.1.  오류|5.4.1.  오류]]
		- [[#5.4.  trouble 4. 커널 프레임 복사 문제 ⭐#5.4.2.  문제 지점 발견|5.4.2.  문제 지점 발견]]
	- [[#5.  문제들 모음 💢#5.5.  trouble 5. 중복 삽입 문제|5.5.  trouble 5. 중복 삽입 문제]]


## 1.  Fork 구현 목적 

- 자식 프로세스를 생성하는 핸들러 구현
- 부모 프로세스의 메모리 공간과 상태를 새로운 공간에 거의 그대로 복제하여 완전히 독립된 새로운 프로세스 만드는 함수 구현

## 2.  fork() 주요 복사 대상 
- VA(가상 주소) 
- 레지스터 상태 
- fd_table 

## 3.  사전 확인 포인트 
1. **동기화 필요성** 
	- 부모 프로세스가 `fork()`호출을 통해 자식 프로세스를 복제해야 한다.
	- 그러나 복제하는 과정은 매우 길고 복잡한 과정이기 때문에 **그 과정과 부모 프로세스의 동기화가 필요할 것**이다.
	- Project 1에서 `cond_variable`를 다뤘던 것처럼 **`semaphore`를 활용해서 동기화를 시킬 예정**
2. **전달되는 값**
	- pintos 코드를 보면 레지스터에 저장되어 커널 영역에 보내지는 변수는 `*thread_name`밖에 없긴 하다 (`pid_t sys_fork(const char *thread_name)`)
	- 이걸로 도대체 뭘 해야 할까??.... 

3. **구현해야 되는 함수** 
	```c
	/* Clones the current process as `name`. Returns the new process's thread id, or
	 * TID_ERROR if the thread cannot be created. */
	tid_t
	process_fork (const char *name, struct intr_frame *if_ UNUSED) {
		/* Clone current thread to new thread.*/
		return thread_create (name,
				PRI_DEFAULT, __do_fork, thread_current ());
	}
		/* A thread function that copies parent's execution context.
	 * Hint) parent->tf does not hold the userland context of the process.
	 *       That is, you are required to pass second argument of process_fork to
	 *       this function. */
	static void
	__do_fork (void *aux) {
		struct intr_frame if_;
		struct thread *parent = (struct thread *) aux;
		struct thread *current = thread_current ();
		/* TODO: somehow pass the parent_if. (i.e. process_fork()'s if_) */
		struct intr_frame *parent_if;
		bool succ = true;
	
		/* 1. Read the cpu context to local stack. */
		memcpy (&if_, parent_if, sizeof (struct intr_frame));
	
		/* 2. Duplicate PT */
		current->pml4 = pml4_create();
		if (current->pml4 == NULL)
			goto error;
	
		process_activate (current);
	```
	- 시스템 콜 핸들러에서 받은 `thread_name`과 `intr_frame`을 활용해서 `process_fork`를 호출할 것 
	- return : 새로운 프로세스 스레드의 id
	- 핵심은 `__do_fork`


## 4.  구현 흐름 공부 

### 4.1.  부모 프로세스가 할 일 
1. 자식에게 전달해 줄 값들을 담을 바구니 만들기 by `malloc`
2. 자식에게 전달할 값 바구니에 담기
3. thread_create 호출하면서 바구니 전달
4. 자식 신호 대기
5. 깬 뒤 자원 정리 

### 4.2.  자식 프로세스가 할 일 
1. 부모가 넘겨준 값 받기
2. 부모의 CPU 레지스터 복사
3. 자식의 페이지 테이블 생성 및 활성화
4. 부모의 가상 주소 공간 복사
5. 부모의 fd 테이블 복사
6. 부모 깨우기
7. 성공 처리

## 5.  문제들 모음 💢

### 5.1.  trouble 1. malloc으로 바구니 만든 점 
처음에는 부모 프로세스가 자식 프로세스에게 넘겨줄 값들을 담은 바구니를 `malloc`으로 메모리 할당 받았다.
하지만 리뷰 과정에서 몇 가지 오류가 있어서 `palloc_get_paeg(0)`로 물리 메모리를 할당받으라고 지적 받았다. <br>
page 단위 메모리 할당으로 바꿔야 하는 이유는 아래와 같다.
1. **안정성**
	- 메모리 할당된 객체는 `fork` 과정에서 **안전하게 다른 스레드에 전달되어야 한다.**
	- `malloc`은 메모리를 쪼개어 사용할 수 있다. 그렇기에 다른 객체들의 주소랑 이웃하는 부분이 많아서 침범 위험이 크다.
	- 반면, 페이지 단위로 할당하는 방식은 **그 페이지 전체가 동일한 객체의 소유이기 때문에 메모리 침범 위험이 적다.**
	  
2. **정렬 보장** 
	- 페이지는 이미 정렬된 상태
	  
3. **pintos 관례** 
	- `fork()`와 무관하게 부팅 로직이 진행될 때 기존 pintos에서 구현된 `process_create_initd`에서 page단위로 메모리를 할당받고 자식 스레드를 생성할 떄 그 값을 넘겨주는 것을 볼 수 있다.
	```c
	tid_t process_create_initd (const char *file_name) {
	  char *fn_copy;
	  ...
	  fn_copy = palloc_get_page (0);  // ✅
		...
	  /* Create a new thread to execute FILE_NAME. */
	  tid = thread_create (file_name, PRI_DEFAULT, initd, fn_copy);
	```
> 페이지 단위 할당은 통째로 + 크게 + 독립적으로 보관되어야 할 때 유리 



### 5.2.  trouble 2. 부모/자식이 fd_table_entry를 공유해버린 버그 

#### 5.2.1.  잘못된 코드 
```c
void do_fork(void *p){
		.....
		
		for (struct list_elem *e = list_begin(&parent->fd_table);
				e != list_end(&parent->fd_table); 
				e = list_next(e)) 
		{
			struct fd_table_entry *entry = list_entry(e, struct fd_table_entry, elem);
			list_push_back(&child->fd_table, &entry->elem);
			// ❌ 잘못된 경우 
		}
		...
```
fork 구현 과정에서 부모의 fd테이블의 내용을 자식에게 복제하는 과정에서 발생한 버그이다. 초기에는 부모의 list_node자체 포인터를 자식 리스트에 바로 공유했다.
이렇게 함으로써 버그가 발생했다. 
**💢문제**
1. 버그 발생 
	- 만약 부모가 file을 close하면 자식이 열어놨던 file도 close가 되어버릴 수 있다. 이는 큰 버그를 발생시킬 수 있음 
2. **수명 관리** 
	- 각자의 fd_table 리스트에 있는 element들은 각자 알아서 수명을 관리하는 것이 좋을 것이다.
	  
> 따라서, **fd_table_entry 객체 공유 금지** 
	  
#### 5.2.2.  해결 코드 ✅ 
```c
static void __do_fork (void *aux) {
		....
		
    for (struct list_elem *e = list_front(&parent->fd_table);
      e != list_end(&parent->fd_table);
      e = list_next(e)) {
      
      struct fd_table_entry *parent_entry = list_entry(e, struct fd_table_entry, elem);
      ....
      // malloc으로 새롭게 메모리 할당 
      struct fd_table_entry *child_entry = malloc(sizeof(struct fd_table_entry));
      ...
      child_entry->fd = parent_entry->fd;
      child_entry->file = file_duplicate(parent_entry->file);
```
`malloc`을 활용해서 새로운 객체를 할당하고 스켈레톤 코드인 `file_duplicate`를 활용해서 파일을 복제하면 된다.


### 5.3.  trouble 3. 가상 페이지 복사 누락 
fork()로직을 작성 후 테스트 코드를 돌려보니 아래와 같은 오류가 발생했다.
```bash
KeyFAIL tests/userprog/fork-once
Kernel panic in run: PANIC at ../../threads/mmu.c:235 in pml4_set_page(): assertion `pg_ofs (kpage) == 0' failed.
Call stack: 0x8004219196 0x800420d781 0x800421cd48 0x800420d097 0x800420d14d 0x800420d1f6 0x800420d292 0x800421ce17 0x80042076d1
Translation of call stack:
0x0000008004219196: debug_panic (lib/kernel/debug.c:32)
0x000000800420d781: pml4_set_page (threads/mmu.c:236)
0x000000800421cd48: duplicate_pte (userprog/process.c:127)
0x000000800420d097: pt_for_each (threads/mmu.c:113 (discriminator 1))
0x000000800420d14d: pgdir_for_each (threads/mmu.c:126 (discriminator 1))
0x000000800420d1f6: pdp_for_each (threads/mmu.c:139 (discriminator 1))
0x000000800420d292: pml4_for_each (threads/mmu.c:152 (discriminator 1))
0x000000800421ce17: __do_fork (userprog/process.c:158 (discriminator 1))
0x00000080042076d1: kernel_thread (threads/thread.c:449)
=> FAIL
=== fork-once session end ===
test 1/1 finish
```
오류 메시지를 보면 `duplicate_pte ` ➡ `pml4_set_page` 이렇게 오류가 전파된 것 같다.근데 나는 `duplicate_pte`를 구현한 적이 없었다.
`duplicate_pte`를 자세히 보니 `#ifdef VM`이 아니라 `ifndef VM`이었다. 즉, Pintos Projcet 3(가상 메모리)을 하기 전에는 이 메서를 구현해야 했던 것이다.
```c
//초기 duplicate_pte
#ifndef VM
/* Duplicate the parent's address space by passing this function to the
 * pml4_for_each. This is only for the project 2. */
static bool
duplicate_pte (uint64_t *pte, void *va, void *aux) {
	struct thread *current = thread_current ();
	struct thread *parent = (struct thread *) aux;
	void *parent_page;
	void *newpage;
	bool writable;

	/* 1. TODO: If the parent_page is kernel page, then return immediately. */

	/* 2. Resolve VA from the parent's page map level 4. */
	parent_page = pml4_get_page (parent->pml4, va);

	/* 3. TODO: Allocate new PAL_USER page for the child and set result to
	 *    TODO: NEWPAGE. */

	/* 4. TODO: Duplicate parent's page to the new page and
	 *    TODO: check whether parent's page is writable or not (set WRITABLE
	 *    TODO: according to the result). */

	/* 5. Add new page to child's page table at address VA with WRITABLE
	 *    permission. */
	if (!pml4_set_page (current->pml4, va, newpage, writable)) {
		/* 6. TODO: if fail to insert page, do error handling. */
	}
	return true;
}
```
코드에 적힌 멘트들을 보면 너무 정리가 잘 되어 있어서 그대로 구현했다.


### 5.4.  trouble 4. 커널 프레임 복사 문제 ⭐
3번까지 해결한 뒤에 아래와 같은 문제가 또 발생했다.


#### 5.4.1.  오류 
```bash

FAIL tests/userprog/fork-once
Kernel panic in run: PANIC at ../../threads/thread.c:276 in thread_current(): assertion `t->status == THREAD_RUNNING' failed.
Call stack: 0x8004219196 0x80042071d4 0x8004206fab 0x8004209f32 0x800421d52b 0x800421e89a 0x800421e712 0x800421e510 0x40003f 0x4000fa 0x400c05
Translation of call stack:
0x0000008004219196: debug_panic (lib/kernel/debug.c:32)
0x00000080042071d4: thread_current (threads/thread.c:278)
0x0000008004206fab: thread_block (threads/thread.c:232 (discriminator 1))
0x0000008004209f32: sema_down (threads/synch.c:75)
0x000000800421d52b: process_wait (userprog/process.c:390)
0x000000800421e89a: sys_wait (userprog/syscall.c:181)
0x000000800421e712: syscall_handler (userprog/syscall.c:103 (discriminator 1))
0x000000800421e510: no_sti (syscall-entry.o:?)
0x000000000040003f: (unknown)
0x00000000004000fa: (unknown)
0x0000000000400c05: (unknown)
FAIL
```
대충 `thread.c`의 thread_current() 문제는 아래의 `do_schedule`코드밖에 없긴하네 
![Pasted image 20250915151742.png](/img/user/supporter/image/Pasted%20image%2020250915151742.png)

- 💢한계 : `do_schedule`에 `breakpoint`를 걸어놨는데 너무 많이 호출돼서 어디서 뭐가 문제인지 찾기 힘들었다.
- ✔**대안** : 좀 더 근본적으로 **`fork`과정에서 언제 do_schedule을 건드릴지 생각해봐야했다.** 그래서 오류 메시지에 뜨는 `t->status == THREAD_RUNNING`보다는 `fork`관련 로직에 breakpoint들을 세세하게 넣고 디버깅을 시도했다.


#### 5.4.2.  문제 지점 발견 
위에서 했던 대안으로 디버깅을 시도한 결과 `fork` 후 `wait`까지 호출되고 현재 thread가 중지된 뒤 이전에 fork했던 스레드 create내부 함수인 `__do_fork`가 실행이 된다. 근데 이 `__do_fork`마지막 `do_iret`명령어가 끝난 뒤 오류가 뜬다. 즉, 자식 프로세스가 유저 모드로 시작 되자마자 터진다는 것. 
✅**유저 모드로 전환 후 문제가 바로 발생했기 때문에 가장 크게 생각해볼만 한 것은 전달하는 레지스터 값을 잘못 준 것 같았다.** 코드를 본 결과 유저 모드 전환 전에 시스템 콜을 통해 전달했던(`syscall_handler(struct intr_frame *f)`) 컨텍스트가 아니라 커널 프레임을 전달하고 있었던 것이다.

**❌기존 코드 문제**
```c
tid_t process_fork (const char *name, struct intr_frame *if_ UNUSED) {
    struct thread *parent = thread_current();
    struct fork_args *args = palloc_get_page(0);
    args->parent = parent;
    args->parent_intr_f = parent->tf;   // ❌ 부모의 커널 컨텍스트를 넘김
    ...
}
```
✅올바른 복사 코드
```c
tid_t process_fork (const char *name, struct intr_frame *if_ UNUSED) {
  ... 
  // 2. 값 채워 넣기
  args->parent = parent;
  args->parent_intr_f = *if_;

...
 static void __do_fork (void *aux) {
  struct intr_frame if_;

  // 1. 부모가 넘겨준 값 받기
  struct fork_args *fork_args = (struct fork_args *) aux;
	...
  struct intr_frame parent_if = fork_args->parent_intr_f;

  // 2. 부모의 CPU 레지스터(유저 컨텍스트) 복사
  memcpy (&if_, &parent_if, sizeof (struct intr_frame));
  current->tf = if_;  
```


### 5.5.  trouble 5. 중복 삽입 문제 

디버깅하면서 4번까지 해결을 했는데도 오류가 떠서 화가 많이 났다. 거의 3시간 동안 이 로직에 디버깅 중인데 또 오류가 터지다니.... 발생한 오류는 아래와 같았다.


```bash
FAIL tests/userprog/fork-multiple
Kernel panic in run: PANIC at ../../lib/kernel/list.c:78 in list_next(): assertion `is_head (elem) || is_interior (elem)' failed.
Call stack: 0x8004219196 0x8004219448 0x800421d6f6 0x800421d5f9 0x800421ea14 0x800421e845 0x800421e623 0x40005a 0x4000e8 0x400159 0x400c64
Translation of call stack:
0x0000008004219196: debug_panic (lib/kernel/debug.c:32)
0x0000008004219448: list_next (lib/kernel/list.c:79)
0x000000800421d6f6: find_child_thread_by_tid (userprog/process.c:412)
0x000000800421d5f9: process_wait (userprog/process.c:391)
0x000000800421ea14: sys_wait (userprog/syscall.c:193)
0x000000800421e845: syscall_handler (userprog/syscall.c:107 (discriminator 1))
0x000000800421e623: no_sti (syscall-entry.o:?)
0x000000000040005a: (unknown)
0x00000000004000e8: (unknown)
0x0000000000400159: (unknown)
0x0000000000400c64: (unknown)
FAIL
```
코드를 보면 `find_child_thread_by_tid` 에서 호출한 list 관련 스켈레톤 코드/함수를 처리하는 과정에서 생긴 문제이다. 오류 메시지 보고 그래도 list관련 문제만 해결하면 될 것 같아서 간단할 것 같았다. 문제는 아래에 있었다.
```c
 static void __do_fork (void *aux) {
	...
	// ❌중복 삽입
  list_push_back(&parent->children, &current->child_elem);
```
- `__do_fork`는 `thread_create`를 하면서 실행되는 함수인데 `thread_create`에서는 이미 자식-부모 관계를 push하면서 관리하고 있다.
- 근데 만약 위의 코드처럼 중복 삽입한다면 리스트 구조가 깨지는 상황이 발생할 수 있고 큰 버그로 이어질 수 있다.
- 따라서 이걸 고치니까 해결 끝 ✅
![Pasted image 20250915220722.png](/img/user/supporter/image/Pasted%20image%2020250915220722.png)