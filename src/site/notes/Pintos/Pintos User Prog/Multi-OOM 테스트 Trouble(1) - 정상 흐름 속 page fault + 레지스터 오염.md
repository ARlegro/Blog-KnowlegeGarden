---
{"dg-publish":true,"permalink":"/Pintos/Pintos User Prog/Multi-OOM 테스트 Trouble(1) - 정상 흐름 속 page fault + 레지스터 오염/","noteIcon":"","created":"2025-12-03T16:03:22.588+09:00","updated":"2025-12-03T16:11:37.840+09:00"}
---


이전 내용 : [[Pintos/Pintos User Prog/Multi-OOM 테스트 코드 심층 분석 및 약간 수정\|Multi-OOM 테스트 코드 심층 분석 및 약간 수정]]

## 1.  문제 상황 - 정상적인 경우에서의 fork 후 page fault발생
- 테스트 코드에서 `wait(pid)`기 음수를 반환 (정상 : 양수)
- 이를 디버그하고자 `printf` + `breakinnpoint`를 활용하였다.

### 1.1.  특이점 발견 1 
> 정상 경로에서 터지는 문제 

디버그를 해보면서 눈에 보였던 점은 **정상적으로 처리됐어야 하는 자식이 비정상적으로 종료가 된 것**이다.
```bash
===========공통인 경우=======
Page fault at 0: not present error writing page in kernel context.
child_9_O: exit(-1)
```
*에러 출력문 해석*
	- **커널 컨텍스트에서 NULL(0)로 쓰기**를 시도하다가 `Page Fault`가 터졌고, 
- 문제 지점은 **“정상 경로(O)”의 child가 종료하는 순간**이다. 
- 정상적인 테스트 흐름에서는 “X” 경로만 죽어야 하는데 “O” 경로에서도 죽은 이유를 추적해야 한다.

생각 : 이 경우 `fork` 자체 로직에 문제가 생긴 것 같다. 디버그 창을 끄고 child_9일 때 fork를 집중적으로 봐야겠다.

### 1.2.  특이점 발견 2 
> 코드 미변경 상태 디버그 ➡ 새로운 버그 발견 

특이점 1을 발견하고 fork를 자세히 뜯어보고자 한 5번은 되돌려보기 한 것 같다. 근데 코드를 아직 바꾸지도 않았는데 갑자기 또 이상한 버그가 생겼다.
```bash
Kernel PANIC at ../../threads/thread.c:273 in thread_current(): assertion `is_thread (t)' failed.
Call stack: 0x80042191a5 0x8004207181 0x8004206c09 0x800421296a 0x8004208e8a 0x80042092e2 0x400038 0x400067 0x400298 0x400487 0x401062.
The `backtrace' program can make call stacks useful.
Read "Backtraces" in the "Debugging Tools" chapter
of the Pintos documentation for more information.
Kernel PANIC recursion at ../../threads/synch.c:228 in lock_acquire().
Interrupt 0x0d (#GP General Protection Exception) at rip=8004222212
 cr2=cccccccccccccce8 error=               0
rax cccccccccccccccc rbx 0000000000000000 rcx 0000008004223bbd rdx 0000000000000031
rsp 000000004747f920 rbp 000000004747f930 rsi 00000000000000e4 rdi cccccccccccccccc
rip 0000008004222212 r8 0000008004223bea  r9 00000080042190c3 r10 0000000000000000
r11 0000000000000212 r12 0000000000008004 r13 2129300000008004 r14 242f000000000000
r15 0000000000008004 rflags 00000002
es: 0010 ds: 0010 cs: 0008 ss: 0010
Interrupt 0x0d (#GP General Protection Exception) at rip=8004221cc4
 cr2=cccccccccccccce0 error=               0
rax cccccccccccccccc rbx 0000000000000000 rcx 0000008004227c40 rdx 000000000000003a
rsp 000000004747f710 rbp 000000004747f720 rsi 000000000000000a rdi cccccccccccccccc
rip 0000008004221cc4 r8 00000080042190c3  r9 000000800421c9db r10 0000000000000000
r11 0000000000000212 r12 0000000000008004 r13 2129300000008004 r14 242f000000000000
r15 0000000000008004 rflags 00000086
es: 0010 ds: 0010 cs: 0008 ss: 0010
Interrupt 0x0d (#GP General Protection Exception) at rip=8004221cc4
 cr2=cccccccccccccce0 error=               0
rax cccccccccccccccc rbx 0000000000000000 rcx 0000008004227c40 rdx 0000000000000003
rsp 000000004747f500 rbp 000000004747f510 rsi 000000000000000a rdi cccccccccccccccc
rip 0000008004221cc4 r8 00000080042190c3  r9 000000800421c9db r10 0000000000000000
r11 0000000000000212 r12 0000000000008004 r13 2129300000008004 r14 242f000000000000
r15 0000000000008004 rflags 00000086
es: 0010 ds: 0010 cs: 0008 ss: 0010
FAIL tests/userprog/no-vm/multi-oom
Kernel panic in run: PANIC at ../../threads/thread.c:273 in thread_current(): assertion `is_thread (t)' failed.
Call stack: 0x80042191a5 0x8004207181 0x8004206c09 0x800421296a 0x8004208e8a 0x80042092e2 0x400038 0x400067 0x400298 0x400487 0x401062
```
`thread.c`파일에서 검증 실패❓
- `thread.c`파일은 사실 project 1에서 끝났기 때문에 project 2인 현재는 만지는 부분이 적었다.(물론 project 1에서 나의 코드가 완벽하지 않을 거라는 점도 염두해 두어야 한다)
- 고민을 하다가 출력창을 자세히 보니 윗 부분 에러 출력창에 `synch.c`에서 `lock_acquire()` 부분에서 문제가 발생한것을 볼 수 있다. 뭘까....? recursion이면 재귀인데? 흠.. 일단 정리해보자 

#### 1.2.1.  에러 창 - `thread_current(): assertion is_thread(t) failed`
- *의미*: 현재 커널 스택의 `t(struct thread *)` 추출이 깨졌거나, thread 구조체의 magic 필드가 덮어써졌다
- *주 원인* - GPT 曰
	1. **커널 스택 오버런**(큰 로컬 변수/재귀/잘못된 memcpy)
	2. **해제된 스택/스레드 구조체에 접근**
	3. **메모리 파괴(이중 free/잘못된 포인터 close 등)**.

```C
 static void __do_fork (void *aux) {
  struct intr_frame if_;

  // 1. 부모가 넘겨준 값 받기
  struct fork_args *fork_args = (struct fork_args *) aux;
  ....
  struct intr_frame parent_if = fork_args->parent_intr_f;
	...
  // 2. 부모의 CPU 레지스터(유저 컨텍스트) 복사
  memcpy (&if_, &parent_if, sizeof (struct intr_frame));
  current->tf = if_;
  
  ...
	current->tf.R.rax = 0;
```
생각을 해보면 fork시에 자식은 부모의 레지스터 값을 복사받는다.
이 부분이 문제일까?? memcpy가??

#### 1.2.2.  에러 창 - `lock_acquire()`에서 패닉 재귀 + 레지스터에 `0xcccccccc...`

**`0xCC` 패턴**
- 이전에 `exec`시스템 콜 트러블 슈팅에서도 보였던 **`0xCC` 패턴**은 **초기화되지 않은 레지스터나 해제된 메모리**를 의미하는 경우가 많다. 

`lock_acuquire()`의 인자는 전역 락으로 사용하고 있었다.
근데 그게 이상하다는 뜻은 **전역 락으로 쓰는 값의 포인터가 이상한 값으로 오염됐다는 뜻** 으로 추측할 수 있다.

`Kernel PANIC recursion at ... in lock_acquire().
- 


*락을 쥔 채로 즉시 yield 시 문제*
- 인터럽트 컨텍스트에서 락을 잡을 일이 많을 것이다. **락을 잡은 상태에서 바로 `thread_yield`를 때리면 큰 문제**가 생긴다
- 

`intr_yield_on_return()`를 사용한다면
- 인터럽트 핸들러가 **작업들을 안정적으로 끝낼 수** 있다 ex. 잡고 있던 락 해제, 스택/레지스터 상태 저장 등
- 이 모든 정리가 끝난 후 스케쥴러에 안정적으로 진입(락 보유 중 중첩 진입이 사라짐)





### 1.3.  상황 정리 
지금 상황에서 여러 에러 로그에서의 핵심 단서는
1. `page_fault` in 커널 컨텍스트
2. `lock_acquire()` 에서 panic 재귀
3. 레지스터에 채워진 쓰레기 값들
4. `thread_current(): assertion is_thread(t) failed`


## 2.  가설 


### 2.1.  1차 가설 
처음에는 그냥 fork로직이 잘못됐나❓ fork할 때 레지스터 값을 이상하게 복사했나❓라고 단순히 생각했다. 

그렇게 고민을 하다가 하루 자고 떠오른 생각이 있었다. 에러가 터지는 타이밍이 신기한데?
- 테스트의 흐름은 (정상 ➡ 비정상 ➡ 정상) 이렇게 테스트를 하는데 **왜 항상 '비정상' 다음의 '정상'흐름에서만 PageFault가 나는것일까**❓
- **첫 번째 정상 흐름에서는 문제가 전혀 발생하지 않았다.**

즉, 일단 생각해볼 점은 비정상 흐름에서 자원 회수 부분이 크게 문제이지 않을까 라고 생각이 들었다.

### 2.2.  조치 1. file_open 내부 
> 페이지 메모리 해제 위치 변경 
```c
int sys_open_file(const char *file){
  strong_validate_addr(file, strlen(file) + 1);
  char *kernel_file = palloc_get_page(0);
  if (kernel_file == NULL){
    return -1;
  }

  strlcpy(kernel_file, file, strlen(file) + 1);
  
  // 락 걸기
  lock_acquire(&filesys_lock);
  struct file *opened_file = filesys_open(kernel_file);
  lock_release(&filesys_lock);
  if (opened_file == NULL){
    return -1;
  }
  // ✅누수
  palloc_free_page(kernel_file);
```
`file`관련된 코드들을 보는데 역시나 누수가 있었다.
페이지를 해제하는 코드의 타이밍이 `opened_file`이 `null`일 경우 전혀 도달하지 않고 있었다. 일단은 이 부분이 원인들 중 하나일 것 같아서 위치변경 
```c
  lock_acquire(&filesys_lock);
  struct file *opened_file = filesys_open(kernel_file);
  lock_release(&filesys_lock);
  // ✅위치 변경 
  palloc_free_page(kernel_file);
  if (opened_file == NULL){
    return -1;
  }
```

### 2.3.  조치 2. FD 테이블 내 FILE CLOSE - 

자원 회수 관련 문제를 집중적으로 코드를 보니`__do_fork`로직애 Leak가 있어 보인다.
- `__do_fork`에서 부모의 FD 테이블 내 `fork`를 **깊은 복사를 하는데 중간에 실패할 경우 이를 회수하는 로직이 없어** 보였다.

*회수 로직을 어디에 추가할까?*
- `__do_fork`에 넣는 것보다 이 프로세스가 error를 만났을 때 `thread_exit() -> process_exit()`을 호출하는데 마지막인 `process_exit()`에서 자원정리 하는 것이 좋아보인다.
- 이렇게 하면 다른 로직에서도 사용되어 효율적인 위치일 거라고 생각이 들어 `process_exit()`에 file 자원 회수를 추가했다.
```c
void process_exit (void) {
  struct thread *cur = thread_current();
  // FD 테이블 전부 닫기
  while (!list_empty(&cur->fd_table)) {
      struct list_elem *e = list_pop_front(&cur->fd_table);
      struct fd_table_entry *entry = list_entry(e, struct fd_table_entry, elem);
      if (entry->file) {
        file_close(entry->file);
      }
      free(entry);
  }
  
  process_cleanup ();
}
```

### 2.4.  조치 3. file 관련된 로직들 전부 lock 일관성 있게 처리

file 관련된 로직들 중 일부는 lock처리하고 일부는 lock 없이 처리하고 했었는데 
락 관련된 것들을 추가해서 좀 더 안정성 있게 해봤다.




### 2.5.  정리 

multi-oom 테스트 코드를 보면 자식들이 두 갈래로 나뉘어 반복 실행된다:
1. *X 경로(의도적 크래시)*:
	- `consume_some_resources_and_die()` 안에서 `NULL`/커널 주소 역참조, **혹은 `open((char*)KERN_BASE)` 같은 “엉터리 포인터”로 `open` 호출**을 던진다. 
	- 이때 **유저 포인터 검증/예외 처리**, **실패 시 자원 회수**가 제대로 돼야 함.
	  
2. *O 경로(정상)*:
	- `consume_some_resources()`만 하고 다른 프로세스들은 살아 있어야 함. 
	- 바로 앞에서 X 경로가 난장판을 친 뒤에도 **FD/락/스케줄러 상태가 일관**해야, 다음 O 경로가 안전하게 돈다.


열어둔 FD 전부 닫기, 이중 close 금지, 락 중첩/재진입 금지


*발생했던 에러들* 
1. `Page fault at 0: ... writing page in kernel context` → **커널이 0 주소에 write 시도**
2. `thread_current(): assertion is_thread(t) failed` → **커널 스택/struct thread 오염** 가능
3. `lock_acquire()` 재귀 + 레지스터에 `0xCC...` → **이미 파괴/해제된 락/구조체 접근** 신호


근데 만약 자원이 회수가 되지 않는다면 
1. 페이지 고갈 -> 
2. ㅇㄹ

#### 2.5.1.  일단 급한 불 해결 

앞서 원인들을 분석한 뒤에 2가지를 보완했다.
1. **메모리를 해제하는 코드 추가**
	- file_open 시 임시로 복사하기 위해 할당한 페이지 메모리를 해제하는 코드 추가 
	  
2. **프로세스/스레드 종료 시 자원 회수 강화**
	- 의도치 않든 의도한 종료일 경우 공통적으로 FD_Table내의 file들을 제대로 정리하고 메모리 해제하는 로직을 추가 

전부다 해결된지는 모르겠다. 하지만 테스트 코드를 다시 실행해보면 이전과 달리 child_10 이상의 콘솔이 찍히며 진행되는 것을 볼 수 있었다.
````bash
테스트 중 pid = 0
===========5이상인 경우=======
Page fault at 0x8004000000: rights violation error writing page in user context.
child_11_X: exit(-1)
===========공통인 경우=======
공통 fork 후 pid = 0
공통 fork 후 pid = 21
부모는 자식(pid = 21)을 기다립니다
테스트 중 i = 12
테스트 중 pid = 0
===========5이상인 경우=======
Page fault at 0: not present error writing page in user context.
child_12_X: exit(-1)
===========공통인 경우=======
공통 fork 후 pid = 0
공통 fork 후 pid = 23
부모는 자식(pid = 23)을 기다립
````

일단은 쉬고 다음에... 너무 힘들었다. 이번 문제들은 0


### 2.6.  중요 ⭐

`multi-oom`의 X 경로는 **커널 주소/NULL 같은 쓰레기 포인터**로 `open` 을 호출한다.
- 위 안전 복사 루틴은 **첫 바이트뿐 아니라 전체 경로를 페이지 단위로 확인**하면서 복사하므로,
- **커널 주소**(is_user_vaddr=false) → 즉시 실패 `-1`
- **NULL** → 첫 바이트는 매핑이라도 **다음 바이트/경계에서 미매핑**이면 실패 `-1`
        
- 즉, **의도적 악성 인자를 `-1` 반환으로 정상 격리**하고, 뒤이어 오는 **정상(O) 경로**에 **오염이 전파되지 않게** 만든다.  
    (이걸 안 하면 `strlen`/`strlcpy`에서 커널 컨텍스트 폴트 → 락/FD/스택 난장 → 바로 다음 O 경로에서 `is_thread`/`0xCC`가 폭발)





### 2.7.  복사 

```c
int sys_open_file(const char *file){
  if (file == NULL || !(is_user_vaddr(file)) || pml4_get_page(thread_current()->pml4, file) == NULL) {
    return -1;
  }
  char *kernel_file = palloc_get_page(0);
  if (kernel_file == NULL){
    return -1;
  }
  ❌
  //strlcpy(kernel_file, file, strlen(file) + 1);
  strlcpy(kernel_file, file, PGSIZE);
```
이거 하나 바꿨다고 원래 pid가 15를 못 넘기던가 89까지 왔다
```bash
===========공통인 경우=======
공통 fork 후 pid = 0
공통 fork 후 pid = 87
부모는 자식(pid = 87)을 기다립니다
테스트 중 i = 45
테스트 중 pid = 0
===========5이상인 경우=======
Page fault at 0x8004000000: rights violation error reading page in user context.
child_45_X: exit(-1)
===========공통인 경우=======
공통 fork 후 pid = 0
공통 fork 후 pid = 89
부모는 자식(pid = 89)을 기다립니다
테스트 중 i = 46
테스트 중 pid = 0
===========5이상인 경우=======
qemu-system-x86_64: QEMU: Terminated vi
```
근데 

앞서 `strlcpy(..., strlen(user)+1)` → `strlcpy(..., PGSIZE)`로 바꾸면서 **버퍼 오버런(커널 메모리 덮어쓰기)** 은 제거했다. 그래서 “초반 즉사”가 사라졌다.  
하지만 **“src 스캔 중 커널 #PF”** 는 여전히 가능해서, **확률적으로 멀리 가다가** 한 번 더 맞으면 그대로 gdbstub 정지로 끝난다.