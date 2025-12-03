---
{"dg-publish":true,"permalink":"/Pintos/Pintos User Prog/Multi-OOM 테스트 Trobule(2) - sema 교체/","noteIcon":"","created":"2025-12-03T14:52:53.071+09:00","updated":"2025-12-03T16:07:47.165+09:00"}
---



이전
- [[Pintos/Pintos User Prog/Multi-OOM 테스트 Trouble(1) - 정상 흐름 속 page fault + 레지스터 오염\|Multi-OOM 테스트 Trouble(1) - 정상 흐름 속 page fault + 레지스터 오염]]
- [[Pintos/Pintos User Prog/Multi-OOM 테스트 코드 심층 분석 및 약간 수정\|Multi-OOM 테스트 코드 심층 분석 및 약간 수정]]

### 0.1.  문제 상황/증상 로그
- 이전 상황들을 분석하고 조치를 했는데도 또 page_fault가 났다.
```yaml
Page fault at 0x2: not present error reading page in kernel context.
...
Call stack: ... intr_handler (...) intr_entry (...)
0x0000008004208031: intr_disable (threads/interrupt.c:152)
0x0000008004207f8d: intr_set_level (threads/interrupt.c:131)
0x000000800420ab08: lock_release (threads/synch.c:384)
...
0x000000800420a200: sema_up (threads/synch.c:146 (discriminator 1))
0x000000800421d16c: __do_fork (userprog/process.c:269)
```
`__do_fork` → `sema_up` → `lock_release` → `intr_set_level` → `intr_disable` (여기서 `Page Fault`)

### 0.2.  분석 흐름 
*✔상황 정리*
- *발생 컨텍스트* : Kernel 모드에서 Page Fault 발생
- *발생 위치* : `__do_fork`에서 부모를 깨우기 위해 `sema_up`을 호출하는 구간
- *추가 단서* : 반복적으로 디버그 실행 시 **레지스터가 오염된 상태로 종료되기도** 함


✅**추측** 
-  `fork`하는 과정에서 부모는 자식의 `fork`과정을 기다려주느라 `semaphore`를 이용한다.
- 근데 그 과정에서 뭔가 문제가 생긴 것처럼 보인다. 
- 특히 이 `page_fault` 오류창 말고도 다시 실행해보면 **레지스터가 오염된 오류창도 떴다.**
- 즉, fork에서 쓰는 lock 메모리를 어디선가 오염시켰고 그거를 그대로 사용하다가 오류가뜬 걸로 추측한다.
- 또한 page_fault는 자식이 종료되면서 **해당 메모리가 해제됐는데 부모가 그 sema를 사용해서 그런거 아닐까?**

## 1.  코드 변경 
* process_fork는 부모 로직
* __do_fork는 자식 로직   

### 1.1.  기존 fork 로직의 문제점 💢
```c
// 부모 
tid_t process_fork (const char *name, struct intr_frame *if_ UNUSED) {
	
	
	sema_down(&args->fork_sema);
  // ❌기존 : 단순히 fork됐는지 확인하고 tid 변경 
  if (child->is_forked == false) { // fork 실패
    tid = TID_ERROR;
  }
  palloc_free_page (args);
  return tid;
}

// 자식 
static void __do_fork (void *aux) {
	...
	if (!succ) {
    goto error;
  }

  // 부모 깨우기 (자식의 fork_sema로 신호)
  sema_up(&current->fork_sema);
  current->tf.R.rax = 0;
  do_iret (&current->tf);

error:
  // 실패 시 부모 꺠우는건 sys_exit에서 알아서 함
  current->is_forked = false;
  sema_up(&current->fork_sema);
  thread_exit();
}
```
*문제*
1. *자식 정리 미흡*
	- semaphore를 이용한 자식 정리 원리는 `process_wait`구현 때 설명했으니 생략하겠다. 
	- 메모리 정리를 엄격하게 테스트하는 `multi-oom`에서 이런 사소한? 메모리 정리를 하지 않으면 나중에 page_fault나 뭐 여러 오류가 떴었다.
	- 실제로 자식 제거 로직의 유무에 따라 테스트의 성공 유무가 결정됐을 정도로 이 테스트를 통과하는데 중요한 부분이다.
2. 

### 1.2.  개선 fork 로직 
#args확장

**✅개선 포인트**
1. *args struct 확장* ⭐
	- `args` 안에 `is_forked`와 `fork_sema`를 넣어 **부모/자식이 동일한 세마포어를 공유하도록 변경** → 자식 스택 안의 세마는 쓰지 않음.
	- 이렇게 되면 부모는 `child`가 먼저 정리되든 안되든 상관없어 좀 더 **안정적인** fork과정을 하도록 했다
2. *hand-shake를 활용한 동기화*
	- *기존* : sema_up만 시키고 무조건 자식 먼저 종료 
	- *개선* : hand-shake과정을 통해 **안정적인 자원 관리** 
	  
3. *`fork` 실패 시 `exit_sema` 재사용 + `sema_init`*
	- 어차피 곧 사라질 자식 → `exit_sema`를 재활용해 안정적으로 종료까지 동기화.
	- 그리고 지금까지 `multi-oom`에서 가끔씩 뜨던 "**락 관련 메모리 오염**" 이런게 너무 짜증나서 `thread_create`할 때 이미 `sema_init`했지만 사용 전에 추가로 init했다.
	  
```c
tid_t process_fork (const char *name, struct intr_frame *if_ UNUSED) {
	// ✅ args의 fork_sema, is_forked 사용 
  sema_down(&args->fork_sema);
  if (args->is_forked == false) { // fork 실패
    tid = TID_ERROR;
    ✅
    list_remove(&child->child_elem);
    sema_up(&child->exit_sema); // 자식도 정리하라고 깨우기
  }
  palloc_free_page (args);
  return tid;
}

// 자식
static void __do_fork (void *aux) {
  ....
  
  // 부모 깨우기 (args 세마로)
  sema_up(&fork_args->fork_sema);
  current->tf.R.rax = 0;
  do_iret (&current->tf);

error:
  fork_args->is_forked = false;
  sema_up(&fork_args->fork_sema);

  // ✅
  sema_init(&current->exit_sema, 0);
  sema_down(&current->exit_sema);
  thread_exit();
}
```
*이 로직의 특징*
1. **세마포어와 플래그 수명 보장**
2. `exit_sema`가 새거다 - 오염되지 않을 가능성이 높음 
3. **메모리 정리 안정화**
	- `hand-shake`를 통한 안전한 메모리 정리(자식 포인터 정리)
4. `fork` 실패 시랑 성공 시랑 부모-자식 처리 순서가 달라진다.
	- `fork` 실패 시 : 자식이 `sema_down`으로 양보해주고 나중에 종료함 
	- `fork` 성공 시 : 자식이 먼저 실행되고 나중에 부모가 실행됨 
	- 뭐 그냥 다르다 이정도?









```BASH
0x0000008004207fc4: intr_enable (threads/interrupt.c:138 (discriminator 1)) 0x0000008004207f8d: intr_set_level (threads/interrupt.c:131) 0x000000800420ab08: lock_release (threads/synch.c:384) 
0x0000008004229b30: (unknown) 0x0000000000000006: (unknown) 0x00000000042070e3: (unknown) 0x000000800420a200: sema_up (threads/synch.c:146 (discriminator 1))
0x000000800421d189: __do_fork (userprog/process.c:274)
```

```bash
===========5이상인 경우=======
Kernel PANIC at ../../threads/thread.c:273 in thread_current(): assertion `is_thread (t)' failed.
Call stack: 0x80042191c3 0x8004207181 0x8004206c09 0x8004212988 0x8004208ea8 0x8004209300 0x400038 0x400067 0x400298 0x4004f3 0x401062.
The `backtrace' program can make call stacks useful.
Read "Backtraces" in the "Debugging Tools" chapter
of the Pintos documentation for more information.
Kernel PANIC recursion at ../../threads/synch.c:228 in lock_acquire().
Interrupt 0x0d (#GP General Protection Exception) at rip=8004222350
 cr2=cccccccccccccce8 error=               0
rax cccccccccccccccc rbx 0000000000000000 rcx 0000008004223cfd rdx 0000000000000038
rsp 000000004747f920 rbp 000000004747f930 rsi 00000000000000e4 rdi cccccccccccccccc
rip 0000008004222350 r8 0000008004223d2a  r9 00000080042190e1 r10 0000000000000000
r11 0000000000000212 r12 0000000000000000 r13 0000000000000000 r14 0000000000000000
r15 0000000000000000 rflags 00000002
es: 0010 ds: 0010 cs: 0008 ss: 0010
Interrupt 0x0d (#GP General Protection Exception) at rip=8004221e02
 cr2=cccccccccccccce0 error=               0
rax cccccccccccccccc rbx 0000000000000000 rcx 0000008004227d80 rdx 0000000000000001
rsp 000000004747f710 rbp 000000004747f720 rsi 000000000000000a rdi cccccccccccccccc
rip 0000008004221e02 r8 00000080042190e1  r9 000000800421c9f9 r10 0000000000000000
r11 0000000000000212 r12 0000000000000000 r13 0000000000000000 r14 0000000000000000
r15 0000000000000000 rflags 00000086
es: 0010 ds: 0010 cs: 0008 ss: 0010
Interrupt 0x0d (#GP General Protection Exception) at rip=8004221e02
 cr2=cccccccccccccce0 error=               0
rax cccccccccccccccc rbx 0000000000000000 rcx 0000008004227d80 rdx 000000000000000a
rsp 000000004747f500 rbp 000000004747f510 rsi 000000000000000a rdi cccccccccccccccc
rip 0000008004221e02 r8 00000080042190e1  r9 000000800421c9f9 r10 0000000000000000
r11 0000000000000212 r12 0000000000000000 r13 0000000000000000 r14 0000000000000000
r15 0000000000000000 rflags 00000086
es: 0010 ds: 0010 cs: 0008 ss: 0010
FAIL tests/userprog/no-vm/multi-oom
Kernel panic in run: PANIC at ../../threads/thread.c:273 in thread_current(): assertion `is_thread (t)' failed.
Call stack: 0x80042191c3 0x8004207181 0x8004206c09 0x8004212988 0x8004208ea8 0x8004209300 0x400038 0x400067 0x400298 0x4004f3 0x401062
Translation of call stack:
0x00000080042191c3: debug_panic (lib/kernel/debug.c:32)
0x0000008004207181: thread_current (threads/thread.c:274)
0x0000008004206c09: thread_tick (threads/thread.c:142)
0x0000008004212988: timer_interrupt (devices/timer.c:151)
0x0000008004208ea8: intr_handler (threads/interrupt.c:351)
0x0000008004209300: intr_entry (intr-stubs.o:?)
0x0000000000400038: (unknown)
0x0000000000400067: (unknown)
0x0000000000400298: (unknown)
0x00000000004004f3: (unknown)
0x0000000000401062: (unknown)
=> FAIL
=== multi-oom session end ===
```


## 2.  포기 

위에 적은 내용들 말고도, Multi OOM 테스트를 하면서 정말 많이 트러블이 났다. 최종적으로는 데스크탑에서 통과가 됐는데 노트북으로는 도저히 Pass가 되지 않았다.
컴퓨터의 차이