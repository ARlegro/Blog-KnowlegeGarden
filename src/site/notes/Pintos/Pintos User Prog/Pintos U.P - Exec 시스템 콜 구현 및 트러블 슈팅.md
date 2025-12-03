---
{"dg-publish":true,"permalink":"/Pintos/Pintos User Prog/Pintos U.P - Exec 시스템 콜 구현 및 트러블 슈팅/","noteIcon":"","created":"2025-12-03T16:03:22.616+09:00","updated":"2025-12-03T16:08:28.641+09:00"}
---


### 0.1.  목차
- 구현 목적
- 트러블 슈팅
- 헷갈렸던 점 
- 교훈

## 1.  구현 목적

*`exec()` 시스템 콜이 하는 일* 
- 새로운 User Program 실행하기 위해 **기존 프로세스의 주소 공간을 교체**하고, **초기 실행 환경을 세팅**하는 기능을 담당

*주요 Point*
- **부모–자식 관계 관리**, **프로세스 생성 시 동기화**, **유저 주소 공간 검증** 같은 운영체제 핵심 개념을 실제 코드로 구현하고 확인하는 것이 주요 목적



## 2.  트러블 슈팅 💢

### 2.1.  Exec-Missing - General Protection Exception 
> push에서 General Protection Exception 

#### 2.1.1.  Preview (미리 보기)
- **현상** : `exec`테스트 코드를 gdb로 디버그 한 결과 stack을 build하는 과정에서 `push()` 호출 시 커널 패닉 발생. 레지스터 값이 `CCCC…` 패턴으로 채워져 있었음.
- **원인** : `load()` 실패 여부를 확인하지 않고 `build_stack()`을 실행하여, 잘못된 인자를 `push`하려다 레지스터 초기화 값(`0xCC` 패턴)을 참조.
- **해결**: `load()` 성공 여부를 먼저 확인한 뒤에만 `build_stack()`을 호출하도록 분기 추가.
- **교훈** : exec은 “로드 성공 → 스택 세팅” 순서를 엄격히 지켜야 하며, 실패 분기 처리가 가장 우선


#### 2.1.2.  현상
- `exec`테스트 코드를 gdb로 디버그 한 결과 stack을 build하는 과정에서 push() 호출 시 커널 패닉 발생. 레지스터 값이 `CCCC…` 패턴으로 채워져 있었음.
```c
void build_stack(struct intr_frame *interrupt_frame, char *argv[], int argc) {
   ....
   for (int i = argc - 1; i >= 0; i--){
	   // ❌
    void *str_addr = push(interrupt_frame, argv[i], (strlen(argv[i]) + 1));
    user_arg_ptrs[i] = (uintptr_t) str_addr;
   }
```

```bash
Interrupt 0x0d (#GP General Protection Exception) at rip=80042216e1
 cr2=cccccccccccccce0 error=               0
rax cccccccccccccccc rbx 0000000000000000 rcx 0000008004227650 rdx 0000000000000025
rsp 00000080042b5660 rbp 00000080042b5670 rsi 000000000000000a rdi cccccccccccccccc
rip 00000080042216e1 r8 00000080042190b4  r9 000000800421c9cc r10 0000000000000000
r11 0000000000000212 r12 000000800421e7d7 r13 0000000000000000 r14 0000000000000000
r15 0000000000000000 rflags 00000086
es: 0010 ds: 0010 cs: 0008 ss: 0010
Interrupt 0x0d (#GP General Protection Exception) at rip=80042216e1
 cr2=cccccccccccccce0 error=               0
rax cccccccccccccccc rbx 0000000000000000 rcx 0000008004227650 rdx 000000000000002e
rsp 00000080042b5450 rbp 00000080042b5460 rsi 000000000000000a rdi cccccccccccccccc
rip 00000080042216e1 r8 00000080042190b4  r9 000000800421c9cc r10 0000000000000000
r11 0000000000000212 r12 000000800421e7d7 r13 0000000000000000 r14 0000000000000000
r15 0000000000000000 rflags 00000086
es: 0010 ds: 0010 cs: 0008 ss: 0010
FAIL tests/userprog/exec-once
Kernel panic in run: PANIC at ../../userprog/exception.c:97 in kill(): Kernel bug - unexpected interrupt in kernel
Call stack: 0x8004219196 0x800421e4f8 0x8004208e7b 0x80042092d3 0x800421d4b2 0x800421d550 0x800421d345 0x800421eb1f 0x800421e913 0x800421e6f6
Translation of call stack:
0x0000008004219196: debug_panic (lib/kernel/debug.c:32)
0x000000800421e4f8: kill (userprog/exception.c:103)
0x0000008004208e7b: intr_handler (threads/interrupt.c:351)
0x00000080042092d3: intr_entry (intr-stubs.o:?)
0x000000800421d4b2: push (userprog/process.c:354)
0x000000800421d550: build_stack (userprog/process.c:371 (discriminator 1))
0x000000800421d345: syscall_exec (userprog/process.c:327)
0x000000800421eb1f: sys_exec (userprog/syscall.c:197)
0x000000800421e913: syscall_handler (userprog/syscall.c:100)
0x000000800421e6f6: no_sti (syscall-entry.o:?)
```

#### 2.1.3.  생각 흐름들 (주저리 주의)
unexpected interrupt in kernel : push를 하면서 커널에서 예상치 못한 인터럽트가 발생한 것 같은데...
근데 `build_stack` 자체는 기존에 잘 사용하고 있던 함수라서 크게 문제 없을 것이다. 우선 생각을 해봐
레지스터에 채워진 값을을 보면 `00000!~~~`은 초기화인 것 같고 근데 rax, rdi에 있는 `ccccc~`이건 뭐지? 검색을 해보니  `CCCC…`는 “**초기화되지 않은 스택/레지스터를 일부 환경에서 채우는 패턴**”으로 유명하다고 한다.⭐
그럼 결국 스택을 `build`하는 과정에서 생긴 문제니까 스택보다는 레지스터를 제대로 초기화하지 않았나? 그리고 테스트 케이스의 제목 자체도 `exec-missing`이기 때문에 잘못된거를 제대로 처리하는지에 대한 테스트일 것이다. 즉, build_stack전에 잘못된 것이 있었을 것이고 그거를 잡아내서 분기처리하는게 이 테스트의 통과 기술이라고 생각을 했고 아래와 같이 바꾸니 통과가 됐다.

```c
// ❌잘못된 패턴 
success = load (argv[0], &_if);
build_stack(&_if, argv, argc);

...
if (!success){
	return TID_ERROR;
}

// ✅ 올바른 패턴 
success = load (argv[0], &_if);
if (!success){
	return TID_ERROR;
}
build_stack(&_if, argv, argc);
```

#### 2.1.4.  해결
**✅출력 결과(다음 에러로)**
```bash
Acceptable output:
  (exec-once) begin
  (exec-once) I'm your father
  (child-simple) run
  exec-once: exit(81)
Differences in `diff -u' format:
  (exec-once) begin
  (exec-once) I'm your father
  (child-simple) run
- exec-once: exit(81)
+ child-simple: exit(81)
FAIL
test 1/1 finish
```
 성공을 확인하는 분기를 추가하니까 일단은 출력값이 뜨기는 했다(근데 기대와 다른 출력값)



### 2.2.  Trouble 2. 기대와 다른 출력값 
 Trouble 1 에서 생긴 문제를 해결했지만 아직 테스트를 통과하지 못했다. 커널 패닉은 발생하지 않았지만 예상과 다른 출력값이 떴기 때문이다.

```bash
Acceptable output:
  (exec-once) begin
  (exec-once) I'm your father
  (child-simple) run
  exec-once: exit(81)
Differences in `diff -u' format:
  (exec-once) begin
  (exec-once) I'm your father
  (child-simple) run
- exec-once: exit(81)
+ child-simple: exit(81)
FAIL
test 1/1 finish
```
#### 2.2.1.  Preview 
- **현상**: 원래는 부모 프로세스가 `exit(81)` 해야 하는데, 자식 프로세스에서 `exit(81)` 로그가 출력됨.
- **원인**: `syscall_exec()` 내부에서 `strlcpy(cur->name, argv[0])`을 실행해 **부모 스레드 이름이 자식 실행 파일 이름으로 바뀜**.
- **해결**: `exec`에서는 부모 스레드의 이름을 바꾸지 않고 그대로 두도록 수정.
- **교훈**: 스레드 이름은 디버깅 보조용일 뿐이지만, 로직을 흐리게 만들어 테스트 실패를 유발할 수 있다.

#### 2.2.2.  생각 흐름 및 해결 
Pintos 운영체제에서 `exit(81)`은 현재 실행 중인 사용자 프로그램을 종료하고, 종료 상태 코드 81을 커널에 반환한다는 뜻
원래는 부모가 종료를 마지막으로 해야되는데 자식에서 종료했다는 뜻. 
디버그를 해보니 일단 부모 쓰레드명으로 계쏙 진행이 되는데 `syscall_exec`에서 `syscall_exec`함수 내부 `strlcpy`부분에서 갑자기 `cur.name`이 바뀌는 것을 봤고 이를 제거하니까 pass~
```c
int syscall_exec(char *command_line){
	....
  build_stack(&_if, argv, argc);
  // ❌이 부분을 삭제해서 child-sim
  // strlcpy(cur->name, argv[0], sizeof(cur->name)); 
```

### 2.3.  Trouble 3. open fail
`exec-missing` 테스트에서 발생한 문제이다.

#### 2.3.1.  Preview
 - **현상**: `exec-missing` 테스트에서 “open failed” 발생.
- **원인**: `load()` 실패 시 `goto done` 후에도 `file_close(file)`을 실행하여, 유효하지 않은 파일 객체를 닫으려 함.
- **해결**: 실행 파일은 프로세스 종료 시점(`process_exit()`/`sys_exit()`)에서만 `file_close()` 하도록 변경.
- **교훈**: 실행 파일 핸들은 특별 관리 대상이며, exec 전용으로 닫아버리면 이후 요구조건(`deny_write` 유지 등)을 맞출 수 없다.

#### 2.3.2.  발생한 문제 
```bash
Putting 'exec-missing' into the file system...
Executing 'exec-missing':
(exec-missing) begin
load: no-such-file: open failed
```
- file

#### 2.3.3.  생각 흐름(주저리)

```c
  // 2. 자식 종료까지 대기

  sema_down(&child->wait_sema);
  int status = child->exit_status;
  list_remove(&child->child_elem);
  // 자식 스레드 깨우기
  sema_up(&child->exit_sema);
  return status;

}
```
부모 스레드가 깨고 실행되는데 여기서 status가 0이 된다???

```c
load (const char *file_name, struct intr_frame *if_) {
	struct thread *t = thread_current ();
	struct ELF ehdr;
	struct file *file = NULL;
	off_t file_ofs;
	bool success = false;
	int i;

	/* Allocate and activate page directory. */
	t->pml4 = pml4_create ();
	if (t->pml4 == NULL)
		goto done;
	process_activate (thread_current ());

	/* Open executable file. */
	file = filesys_open (file_name);
	if (file == NULL) {
		printf ("load: %s: open failed\n", file_name);
		goto done;
	}


done:
	/* We arrive here whether the load is successful or not. */
	file_close (file); // ❌
	return success;
}
```
`LOAD`에서 `goto done`으로 분기 처리가 되면  done으로 가서 `FILE`을 close하는게 문제다. 이것 때문에 `exec` 요구조건을 못 맞춘 상태. 

#### 2.3.4.  해결 
>[!QUESTION] 어떻게 해결해야할까❓
- 일단, `exec`전용 `load`함수를 따로 만드는건 비효율적이라고 생각했다.
 - 따라서 `file`닫는거는 스레드가 종료될 때(`thread_exit()`)에서 관리하는 걸로 ㄱ 

`process_execute` 로직에서 문제 
```c
  build_stack(&_if, argv, argc);

  // strlcpy(cur->name, argv[0], sizeof(cur->name));

  palloc_free_page (command_line);

  // cur->exit_status = 81;

  // ❌ 잘못된 것 sema_up(&cur->wait_sema);
	/* 부모는 '자식 종료' 시 sys_exit()에서 깨어나야 한다 */
  do_iret (&_if);
```
- `exec`이면 부모는 깨우는게 아니다.
- GPT 曰 : exec 핸들러 구현 시 `sys_exit(int status)` 안에서만 부모를 깨워라. (자식 **진짜 종료 시점**)


### 2.4.  Trouble 4. rox 문제 

#### 2.4.1.  Preview
- **현상**: 자식 종료 후 부모가 실행 파일에 `write()` 시도 시 실패. 기대 동작은 **성공**이어야 함.
- **원인**: `sys_exit()`에서 부모를 `sema_up()`으로 너무 빨리 깨워, 부모가 쓰기를 시도할 때 자식 실행 파일의 `deny_write`가 아직 해제되지 않음.
- **해결**: `file_allow_write()` 및 `file_close()`를 **부모 깨우기 이전**에 실행되도록 순서 조정.
- **교훈**: 동기화 순서가 곧 요구사항 충족 여부를 결정한다. “부모 깨우기”는 항상 **자식 정리 완료 이후**여야 한다.
#### 2.4.2.  문제 상황 
✔요구사항 스펙 
```bash
(rox-child) begin
(rox-child) open "child-rox"
(rox-child) read "child-rox"
(rox-child) write "child-rox"
(rox-child) exec "child-rox 1"
(child-rox) begin
(child-rox) try to write "child-rox"
(child-rox) try to write "child-rox"
(child-rox) end
child-rox: exit(12)
(rox-child) write "child-rox"
(rox-child) end
rox-child: exit(0)
```

💢내 로그 
```bash
Executing 'rox-child':
(rox-child) begin
(rox-child) open "child-rox"
(rox-child) read "child-rox"
(rox-child) write "child-rox"
(rox-child) exec "child-rox 1"
(child-rox) begin
(child-rox) try to write "child-rox"
(child-rox) try to write "child-rox"
(child-rox) end
child-rox: exit(12)
(rox-child) write "child-rox"
(rox-child) write "child-rox": FAILED
rox-child: exit(1)
Execution of 'rox-child' complete.
```
테스트 `rox-child`의 마지막 단계에서 부모 프로세스가 `child-rox`에 `write()`를 시도하면 **FAILED**가 발생.
기대 동작 : 자식 종료 시 **`deny_write`가 해제**되어 부모가 **다시 쓰기가 성공**해야 함.

#### 2.4.3.  문제 원인 
- `sys_exit`함수 로직을 탈 때 부모를 깨우는 시점이 실행 파일 잠금 해제보다 빠름 
```c
void sys_exit(int status){

	struct thread *cur = thread_current();
	cur->exit_status = status;
	printf("%s: exit(%d)\n", cur->name, status);

	
	if (cur->parent != NULL){
		// sema_up(&cur->wait_sema);  // 너무 이른 깨움 💢
		if (cur->running_file){
			file_close(cur->running_file);
			cur->running_file = NULL;
		}

		sema_up(&cur->wait_sema);  // 올바른 꺠움
		sema_down(&cur->exit_sema);
	}
	
	thread_exit();
}
```
기존
- `process_exit()`에서 `running_file`에 대한 `file_allow_write`, `file_close()`를 처리해서 이전에 부모가 깨고 running file을 읽어버리는 시도를 했었다.
- 예외 처리를 다 해놓아서 `(rox-child) write "child-rox": FAILED`는 됐지만 애초에 읽지도 못하게 만들어야 했다

즉, 부모의 `wait(child)`가 **일찍** 반환되어, 부모가 `write()`를 시도하는 시점에 자식 쪽의 `deny_write`가 아직 **해제 전**이라 실패.


### 2.5.  cannot access - exec

```c
tid_t syscall_process_execute (const char *file_name) {
  char *fn_copy;
  tid_t tid;
  fn_copy = palloc_get_page (0);
  if (fn_copy == NULL){
    return TID_ERROR;
  }
  strlcpy (fn_copy, file_name, PGSIZE);
```


### 2.6.  Trouble 5. exec-read (파일 오프셋 공유 문제)

#### 2.6.1.  Preview 
- **현상**: 부모·자식이 같은 파일을 읽을 때, 오프셋이 공유되지 않고 각각 독립적으로 움직여 테스트 실패.
- **원인**: 단순히 `file_reopen()`으로 복제하면 inode는 공유하지만 오프셋은 별개로 관리됨. POSIX는 **open file description을 공유**하도록 요구.
- **해결**: `file_duplicate()` 함수를 구현해 참조 카운트 기반으로 **같은 `struct file` 객체를 공유**하도록 변경.
- **교훈**: fork/exec 시 파일 디스크립터는 단순 복사가 아니라 **공유语의 복제**를 해야 POSIX 호환 동작이 보장된다.

#### 2.6.2.  문제 현상
기대와 다른 출력 - exec-read
```bash
hd1: unexpected interrupt
hd1:0: detected 274 sector (137 kB) disk, model "QEMU HARDDISK", serial "QM00003"
Formatting file system...done.
Boot complete.
Putting 'exec-read' into the file system...
Putting 'sample.txt' into the file system...
Putting 'child-read' into the file system...
Executing 'exec-read':
(exec-read) begin
(exec-read) open "sample.txt"
(exec-read) read "sample.txt" first 20 bytes
(child-read) begin
(child-read) open "sample.txt"
(child-read) read "sample.txt" first 20 bytes
(child-read) read "sample.txt" remainders
(child-read) expected text:

"KAIST is the first and top science and technology university in Korea.
 KAIST has been the gateway to advanced science and technology,
 innovation, and entrepreneurship, and our graduates have been key
 players behind Korea’ innovations. KAIST will continue to pursue
 advances in science and technology as well as the economic development
 of Korea and beyond." --KAIST

(child-read) text actually read:
"KAIST is the first "KAIST is the first and top science and technology university in Korea.
 KAIST has been the gateway to advanced science and technology,
 innovation, and entrepreneurship, and our graduates have been key
 players behind Korea’ innovations. KAIST will continue to pursue
 advances in science and technology as well as the economic development
 of Korea 
(child-read) expected text differs from actual: FAILED
```

#### 2.6.3.  시도 
`file_reopen()`이 새 `struct file`을 만들어서 같은 inode를 가리키게 해줌
```c
struct file *file_reopen (struct file *file) {
  return file_open (inode_reopen (file->inode));
```
```c
// fork 과정 중 
child_entry->file = file_reopen(parent_entry->file);
file_seek (child_entry->file, file_ofs);
```
`file_reopen()`으로 새로운 file 객체를 받고 `file_seek()`으로 `offset`을 0으로 초기화 했다(부모와 `offset`이 다름)


💢 문제 
```bash
(child-read) expected text:
"KAIST is the first and top science and technology university in Korea.
 KAIST has been the gateway to advanced science and technology,
 innovation, and entrepreneurship, and our graduates have been key
 players behind Korea’ innovations. KAIST will continue to pursue
 advances in science and technology as well as the economic development
 of Korea and beyond." --KAIST
(child-read) text actually read:
"KAIST is the first "KAIST is the first and top science and technology university in Korea.
 KAIST has been the gateway to advanced science and technology,
 innovation, and entrepreneurship, and our graduates have been key
 players behind Korea’ innovations. KAIST will continue to pursue
 advances in science and technology as well as the economic development
 of Korea
```

`child-read` 류 테스트는 **fork 후 부모·자식이 같은 “열림 상태(open file description)”를 공유**해서, 한쪽이 20바이트 읽으면 **다른 쪽의 오프셋도 20**이 되는 POSIX 스타일을 기대
즉, 스냅샷 복사가 아니라 **공유 오프셋** 방식으로 설계를 했어야 했다.

#### 2.6.4.  해결 - `file_duplicate(parent_file)` 사용
**공유 오프셋 복제 함수**로 바꿔: `file_duplicate(parent_file)` 사용하여 내부적으로 **참조 카운트(refcount)** 를 두고, **같은 `pos`를 공유**하도록 설계.




## 3.  부팅 시 exec과 유저 시스템 콜 EXEC의 차이 헷갈린 점 

### 3.1.  부팅 시 exec
- 호출 대상 : 커널 스레드. `process_exec`을 호출해서 처리한다.
- 현재 프로세스의 주소 공간을 비우고 tokenize, load, build_stack 등을 한뒤 바로 해당 스레드를 유저 모드로 jump하는(`do_iret`) 로직이였다.
- 이는 exec과정이 성공해도 반환값이 없었다.
`
### 3.2.  유저 시스템 콜 exec
- 호출 대상 : 유저 프로그램 -> 시스템 콜 핸들러 -> 부모 프로세스 컨텍스트 
- 부팅 시 exec과 달리 인자가 유저 가상 주소이다 ➡ 따라서 커널 페이지로 안전 복사 必

부팅 시 `exec`과 크게 다른 점 
- 자식 스레드 생성 작업을 하면서 부모는 기다린다.
- 자식 스레드 생성이 성공하면 자식 pid를 반환하고 실패하면 -1 반환 

## 4.  교훈 
1. **성공/실패 분기 처리**를 명확히 해야 한다. (`load` → `stack` 순서).
2. **이름, 핸들 등 보조 정보 수정**도 전체 실행 흐름에 큰 영향을 줄 수 있다.
3. **파일 핸들 관리**는 `exec`에서 가장 까다롭다. `deny_write` 타이밍과 오프셋 공유는 운영체제의 세밀한 동작 방식을 반영한다.
4. 결국 `exec` 구현은 단순 “프로그램 실행”이 아니라 **부모–자식 동기화, 파일시스템 제약, POSIX semantics**를 모두 아우르는 종합 과제였다.

