---
{"dg-publish":true,"permalink":"/Pintos/Pintos User Prog/Pintos U.P - 사전 분석/","noteIcon":"","created":"2025-12-03T16:03:22.675+09:00","updated":"2025-12-03T16:03:22.682+09:00"}
---


> [!info] Pintos를 시작하면서 User Program은 도대체 어떤 Flow인지 정도만 이해하기 위해 글 작성 (시스템 콜 자세히는 X)

## 1.  User Program은 뭘까?
> 부팅 이후, 사용자 프로그램을 메모리에 로드하고 실행하기 위한 기능 모듈 


최초 실행 흐름 
```scss 
init.c → run_actions → run_task(argv)
           → process_wait(process_create_initd(task))
```

- `process_create_initd` 
	- 최초의 유저 프로그램을 새 스레드로 생성해서 로드하고 진입 ⭐
	- 새 stack / 새 페이지 테이블 / 파일 로드 준비 
	- 내부적으로 `exec()` 호출하여 실제로 ELF 로딩하고 사용자 모드로 진입한다
	  
- `process_wait(tid)`
	- 부모가 자식을 기다려 종료 코드를 회수 


>[!tip] 핵심
>- 수명 관리 : 부팅 시 기존 스레드(프로세스)와 새로 부팅하려는 User Program생성용 스레드의 수명 관리
>- 새로운 User Program을 만들 때의 Loading,Stack만들기, 인자 전달이 핵심 

## 2.  User Program을 위한 OS의 역할 
OS는 UserProgram이 정상적으로 실행되고 종료되기 위해 아래와 같은 담당을 한다.
1. 프로세스 생성 : 새로운 User Program을 실행하기 위해 
2. ELF 실행 파일 파싱 & Load
3. User Stack 생성
4. Argument Passing - 프로그램 인자(argv, argc) 전달
5. 프로세스 종료(wait, exit) 처리
6. System Call 지원

> User Program 실행을 위해 OS는 사용자 공간을 만들고, ELF를 올리고, 스택을 구성해 CPU를 사용자 모드로 넘겨준다.

## 3.  User Program 프로젝트 목표 

*🥊이전 Project 1*
- 이전 Project 1은 운영체제가 사용자 프로그램으로부터 제어권을 다시 가져오는 방법인 **타이머와 I/O 장치로부터의 외부 인터럽트**를 다뤘다.

*✅Project 2 - User Program*
Project 2는
1. 컴퓨터 부팅 시 사용자 프로그램 올바르게 실행시키고
2. 그 프로그램이 OS와 상호작용할 수 있도록 기능 구현하는게 목표

*(사전 학습 : 동기화, 시스템콜, 가상 주소)*



## 4.  Process_wait

### 4.1.  어디서 등장?
`init.c -> run_actions -> run_task`에서 run_task 함수 내에서 불러지는 함수이다.

```c
static void
run_task (char **argv) {
  const char *task = argv[1];
  printf ("Executing '%s':\n", task);

#ifdef USERPROG
  if (thread_tests){
    run_test (task);
  } else {
    process_wait (process_create_initd (task)); 
  }
```

### 4.2.  💢현재 문제 - Pintos 기본 구현은 텅 비어 있음 
- `process_excute`를 호출시키는 `process_wait`의 함수가 거의 아무것도 채워져 있지 않다
```c
int process_wait (tid_t child_tid UNUSED) {
  /* XXX: Hint) The pintos exit if process_wait (initd), we recommend you
   * XXX:       to add infinite loop here before
   * XXX:       implementing the process_wait. */
  return -1;
}
```

>[!QUESTION] 뭘 해야 할까?
>**자식 프로세스의 완료를 기다리도록 만들어서** ➡ 프로세스가 제대로 실행되고 종료될 수 있도록 하는 것

## 5.  프로그램 실행의 전반적인 구조 파악
Pintos UserProgram 시작 부분 핵심 Flow(system call 제외)
```scss 
process_create_initd()
 → thread_create()로 child 생성
    → child가 시작되면 start_process() 진입
        → process_exec()
             → load()
                → setup_stack()
                → argument passing
             → do_iret() → user mode 진입
```


### 5.1.  process_exec 
#새로운Load  

> **이미 실행 중인 프로세스(스레드) 안에서 새로운 User Program을 로드해서 실행**하는 역할
> 기존 주소 공간을 폐기하고 새로운 User Program으로 교체하는 작업 


흐름
1. 기존 주소 공간 폐기 - `process_cleanup`
2. 새 주소 공간/PML4 준비 
3. `load()`
4. 성공 시 `do_iret()` 으로 사용자 모드 진입

```c
tid_t process_create_initd (const char *file_name) {
  char *fn_copy;
  tid_t tid;
  process_init ();

	// 불려지는 곳 
  if (process_exec (f_name) < 0)
    PANIC("Fail to launch initd\n");
  NOT_REACHED ();
}  
```
```c
int process_exec (void *f_name) {
	// 프로그램 파일명 
	char *file_name = f_name;
	bool success;

	/* We cannot use the intr_frame in the thread structure.
	 * This is because when current thread rescheduled,
	 * it stores the execution information to the member. */
	   
	// 새 주소 공간 생성 - 초기화 必
	struct intr_frame _if;
	_if.ds = _if.es = _if.ss = SEL_UDSEG;
	_if.cs = SEL_UCSEG;
	_if.eflags = FLAG_IF | FLAG_MBS;

	/* We first kill the current context */
	process_cleanup ();

	/* And then load the binary */
	// ✅ load
	success = load (file_name, &_if);

	/* If load failed, quit. */
	palloc_free_page (file_name);
	if (!success)
		return -1;

	/* Start switched process. */
	// 성공 시 사용자 모드로 점프 
	do_iret (&_if);
	NOT_REACHED ();
}
```

`load()` ⭐
- **바이너리 파일을 메모리로** 올린다.
- 프로세스에 필요한 **페이지 테이블 생성**
- 파일의 ELF헤더를 읽어 데이터와 텍스트 세그먼트를 메모리에 올리고
- 사용자 스택을 생성 및 초기화 

### 5.2.  Load() 과정 상세 

#### 5.2.1.  입력이 들어온다.
```C
static bool
load (const char *file_name, struct intr_frame *if_) {
```
- `file_name` : 실행할 ELF 바이너리 경로
- `if_` : 새 유저 프로세스가 처음 진입할 때 사용할 레지스터 모음 (ex. `rip`, `rsp`)

#### 5.2.2.  주소 공간 준비 (PML4 생성 및 활성화)
커널은 주어진 스레드를 위한 page table을 생성
- **프로세스마다 독립적**인 `pml4`가 필요 (새로운 주소 공간)
- 4단계 페이징 구조의 최상위 Level 테이블을 만드는 것 

```c
// load() 내 흐름들 
t->pml4 = pml4_create (); // 새로운 프로세스 주소 공간 생성.
if (t->pml4 == NULL){
		goto done;
}
process_activate (thread_current ());		


// CREATE 함수 
pml4_create (void) {
	uint64_t *pml4 = palloc_get_page (0);
	if (pml4)
			 // base_pml4(커널 기본 페이지 맵)을 복사해서 커널 공간 매핑은 유지
			memcpy (pml4, base_pml4, PGSIZE);
	return pml4;
}
```
- `pml4_create()`
    - 새 페이지(4KB)를 하나 할당해 **프로세스 전용 PML4**를 만든다.
    - 커널이 항상 접근 가능한 **커널 매핑(base_pml4)** 를 그대로 복사해 둔다(커널 공간 공유).
    
- `process_activate()`
    - CPU의 **CR3**를 현재 스레드의 `PML4`로 바꿔 **해당 주소 공간을 활성화**한다.
	    - CPU가 메모리에 접근할 때 새로운 `PML4`를 기준으로 주소를 변환함 
	    - CR3 = CPU가 "**현재 어떤 주소 공간을 쓰는지**" 알려주는 레지스터 
    - 커널 스택 포인터 등도 스레드에 맞춰 갱신
    - *만약 비활성하면❓Page Fault 발생* 
	    - thread는 page table 주소값을 가지고 있고, page table은 'data'영역, 'BSS영역' 등을 알고 있다.



#### 5.2.3.  실행 파일 열기 
```c
file = filesys_open(file_name);
if (file == NULL) {
	 printf("open failed\n");
	 goto done; 
}
```
사용자 프로그램 파일을 연다. 실패 시 종료 경로.

#### 5.2.4.  ELF 파싱 & 세그먼트 순회 및 매핑 

자세한 내용은 Project 3(VM)에서 배울 것 



#### 5.2.5.  유저 스택 생성 
`setup_stack(if_)`
- 유저 상단 주소(예: `USER_STACK` 근처)에 **1페이지**를 할당해 **초기 스택 프레임**을 만든다.
- `setup_stack()`을 통해 `if_->rsp`를 **스택 최상단**으로 설정
- 구현 해야하는 것
	- 스택을 채워나가야 한다.
	- Pintos에서는 UserProgram을 실행하기 위한 Stack이 완성되어 있지 않다.



#### 5.2.6.  Argument Passing - 인자 전달 필요 ⭐

> Pintos에서 인자 전달 : `argv[]`와 `argc`를 스택에 올려주는 기능 
> - `main(argc, argv)` 형태로 유저 프로그램이 실행될 수 있도록  
> - argv 문자열들을 user stack 위로 `push`해야 한다.


```swift
"echo hello world" → split → ["echo", "hello", "world"]
스택 구조:

| "echo\0" |
| "hello\0" |
| "world\0" |
| argv[3] = NULL |
| argv[2] ptr |
| argv[1] ptr |
| argv[0] ptr |
| argc |
| return address (0) |
```

- **argument passing 기능**: 
	- 문자열을 잘라(`strtok_r` 같은 걸로 공백 단위 split) `argv[0] = "echo"`, `argv[1] = "x"`, `argv[2] = "y"`, `argv[3] = "z"` 형태로 스택에 쌓아주고, 
	- `argc`와 `argv` 포인터를 세팅해서 사용자 프로그램이 `main(int argc, char **argv)`로 받을 수 있도록 해줘야 한다.


Pintos에서는 기본적으로 process_exec을 위해 command line 전체를 전달해준다
![Pasted image 20250912012637.png](/img/user/supporter/image/Pasted%20image%2020250912012637.png)
토큰화 해서 thread_create() 로직 시 쓰레드 명 전달하도록 


#### 5.2.7.  로딩 후 모습 
> OS는 메모리에 올라온 프로그램 파일을 읽는다.

프로세스는 이제:

- 텍스트/데이터 세그먼트 매핑
- 사용자 스택 준비 완료
- rip = user_entry (ELF entry point)
- rsp = pass된 스택 top
    
상태에서 user mode로 진입할 준비가 완료됨.

[[Pintos/Pintos VM/Pintos VM - 스택 개념 및 구조\|Pintos VM - 스택 개념 및 구조]]