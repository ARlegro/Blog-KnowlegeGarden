---
{"dg-publish":true,"permalink":"/Pintos/Pintos User Prog/Multi-OOM 테스트 코드 심층 분석 및 약간 수정/","noteIcon":"","created":"2025-12-03T16:03:22.597+09:00","updated":"2025-12-03T16:07:32.830+09:00"}
---



### 0.1.  목차
- [[#1.  테스트 케이스 분석|1.  테스트 케이스 분석]]
	- [[#1.  테스트 케이스 분석#1.1.  Preview : 테스트 케이스 전체|1.1.  Preview : 테스트 케이스 전체]]
	- [[#1.  테스트 케이스 분석#1.2.  test파일 일부 1. 주석 확인|1.2.  test파일 일부 1. 주석 확인]]
		- [[#1.2.  test파일 일부 1. 주석 확인#1.2.1.  주석 및 분석|1.2.1.  주석 및 분석]]
		- [[#1.2.  test파일 일부 1. 주석 확인#1.2.2.  체크 포인트|1.2.2.  체크 포인트]]
	- [[#1.  테스트 케이스 분석#1.3.  핵심 로직 1. main|1.3.  핵심 로직 1. main]]
	- [[#1.  테스트 케이스 분석#1.4.  핵심 로직 2. make_children|1.4.  핵심 로직 2. make_children]]
		- [[#1.4.  핵심 로직 2. make_children#1.4.1.  코드 및 분석|1.4.1.  코드 및 분석]]
		- [[#1.4.  핵심 로직 2. make_children#1.4.2.  체크 포인트|1.4.2.  체크 포인트]]
	- [[#1.  테스트 케이스 분석#1.5.  핵심 로직 3. consume_some_resources()|1.5.  핵심 로직 3. consume_some_resources()]]
		- [[#1.5.  핵심 로직 3. consume_some_resources()#1.5.1.  코드 및 분석|1.5.1.  코드 및 분석]]
		- [[#1.5.  핵심 로직 3. consume_some_resources()#1.5.2.  체크 포인트|1.5.2.  체크 포인트]]
	- [[#1.  테스트 케이스 분석#1.6.  핵심 로직 3. make_children 내부 분기 로직|1.6.  핵심 로직 3. make_children 내부 분기 로직]]
		- [[#1.6.  핵심 로직 3. make_children 내부 분기 로직#1.6.1.  코드 및 분석|1.6.1.  코드 및 분석]]
		- [[#1.6.  핵심 로직 3. make_children 내부 분기 로직#1.6.2.  체크 포인트|1.6.2.  체크 포인트]]
	- [[#1.  테스트 케이스 분석#1.7.  핵심 로직 4. consume_some_resources_and_die|1.7.  핵심 로직 4. consume_some_resources_and_die]]
		- [[#1.7.  핵심 로직 4. consume_some_resources_and_die#1.7.1.  이 함수 호출한 주체의 의도|1.7.1.  이 함수 호출한 주체의 의도]]
		- [[#1.7.  핵심 로직 4. consume_some_resources_and_die#1.7.2.  코드 및 분석|1.7.2.  코드 및 분석]]
	- [[#1.  테스트 케이스 분석#1.8.  중간 점검|1.8.  중간 점검]]
	- [[#1.  테스트 케이스 분석#1.9.  wait 로직 초기화|1.9.  wait 로직 초기화]]
- [[#2.  트러블 슈팅으로|2.  트러블 슈팅으로]]


## 1.  테스트 케이스 분석 

### 1.1.  Preview : 테스트 케이스 전체 

메서드 종류
1. `make_children(void)`
2. `consume_some_resources(void)`
3. `consume_some_resources_and_die (void)`
4. `int make_children (void)`
5. `int main (int argc UNUSED, char *argv[] UNUSED)`
```c
/* Recursively forks until the child fails to fork.
   We expect that at least 28 copies can run.
   
   We count how many children your kernel was able to execute
   before it fails to start a new process.  We require that,
   if a process doesn't actually get to start, exec() must
   return -1, not a valid PID.

   We repeat this process 10 times, checking that your kernel
   allows for the same level of depth every time.

   In addition, some processes will spawn children that terminate
   abnormally after allocating some resources.

   We set EXPECTED_DEPTH_TO_PASS heuristically by
   giving *large* margin on the value from our implementation.
   If you seriously think there is no memory leak in your code
   but it fails with EXPECTED_DEPTH_TO_PASS,
   please manipulate it and report us the actual output.
   
   Orignally written by Godmar Back <godmar@gmail.com>
   Modified by Minkyu Jung, Jinyoung Oh <cs330_ta@casys.kaist.ac.kr>
*/


static const int EXPECTED_DEPTH_TO_PASS = 10;
static const int EXPECTED_REPETITIONS = 10;

int make_children (void);

static void consume_some_resources (void)
{
  int fd, fdmax = 126;

  /* Open as many files as we can, up to fdmax.
	 Depending on how file descriptors are allocated inside
	 the kernel, open() may fail if the kernel is low on memory.
	 A low-memory condition in open() should not lead to the
	 termination of the process.  */
  /**
   * 가능한 많은 파일들을 열돼 
   * 커널 내부에서 파일 디스크립터가 어떻게 할당되는지에 따라, open()이 실패할 수도 있다
   * ex. 테이블 크기, 내부 자료구조, 메모리 부족 등
   * open()에서 low-memory(메모리 부족) 상황이 발생하더라도, 그 자체가 프로세스 종료로 이어지면 안 된다
   * 즉, open실패가 단순히 open() 호출이 실패한 것으로 처리해야 한다는 말
   * 목적
   * 1. 커널이 많은 수의 파일을 열 수 있도록 허용하는지
   * 2. 만약 자원 부족으로 커널 open()이 실패해도 그걸 정상적인 동작으로 간주하고 프로세스를 살리는지 
   */
  for (fd = 0; fd < fdmax; fd++) {
#ifdef EXTRA2
	  if (fd != 0 && (random_ulong () & 1)) {
		if (dup2(random_ulong () % fd, fd+fdmax) == -1)
			break;
		else
			if (open (test_name) == -1)
			  break;
	  }
#else
		if (open (test_name) == -1)
		  break;
#endif
  }
}

/* Consume some resources, then terminate this process
   in some abnormal way.  */
static int NO_INLINE
consume_some_resources_and_die (void)
{
  consume_some_resources ();
  int *KERN_BASE = (int *)0x8004000000;

  switch (random_ulong () % 5) {
	case 0:
	  *(int *) NULL = 42;
    break;

	case 1:
	  return *(int *) NULL;

	case 2:
	  return *KERN_BASE;

	case 3:
	  *KERN_BASE = 42;
    break;

	case 4:
	  open ((char *)KERN_BASE);
	  exit (-1);
    break;

	default:
	  NOT_REACHED ();
  }
  return 0;
}

int
make_children (void) {
  int i = 0;
  int pid;
  char child_name[128];
  for (; ; random_init (i), i++) {
    printf("테스트 중 i = %d", i);
    if (i > EXPECTED_DEPTH_TO_PASS/2) {
      snprintf (child_name, sizeof child_name, "%s_%d_%s", "child", i, "X");
      pid = fork(child_name);
      if (pid > 0 && wait (pid) != -1) {
        fail ("crashed child should return -1.");
      } else if (pid == 0) {
        consume_some_resources_and_die();
        fail ("Unreachable");
      }
    }

    snprintf (child_name, sizeof child_name, "%s_%d_%s", "child", i, "O");
    pid = fork(child_name);
    if (pid < 0) {
      exit (i);
    } else if (pid == 0) {
      consume_some_resources();
    } else {
      break;
    }
  }

  int depth = wait (pid);
  if (depth < 0)
	  fail ("Should return > 0.");

  if (i == 0)
	  return depth;
  else
	  exit (depth);
}

int
main (int argc UNUSED, char *argv[] UNUSED) {
  test_name = "multi-oom";

  msg ("begin");

  int first_run_depth = make_children ();
  CHECK (first_run_depth >= EXPECTED_DEPTH_TO_PASS, "Spawned at least %d children.", EXPECTED_DEPTH_TO_PASS);

  for (int i = 0; i < EXPECTED_REPETITIONS; i++) {
    int current_run_depth = make_children();
    if (current_run_depth < first_run_depth) {
      fail ("should have forked at least %d times, but %d times forked", 
              first_run_depth, current_run_depth);
    }
  }

  msg ("success. Program forked %d iterations.", EXPECTED_REPETITIONS);
  msg ("end");
}

```


### 1.2.  test파일 일부 1. 주석 확인 
#### 1.2.1.  주석 및 분석 
```c
/* 
	 // 1. 반복적 fork
	 Recursively forks until the child fails to fork.
   We expect that at least 28 copies can run.
   
   // 2. exec() 반환 값
   We count how many children your kernel was able to execute
   before it fails to start a new process.  We require that,
   if a process doesn't actually get to start, exec() must
   return -1, not a valid PID.

	 // 3. 반복
   We repeat this process 10 times, checking that your kernel
   allows for the same level of depth every time.

	 // 4. 비정상 프로세스 생성 
   In addition, some processes will spawn children that terminate
   abnormally after allocating some resources.

	 // 5. 
   We set EXPECTED_DEPTH_TO_PASS heuristically by
   giving *large* margin on the value from our implementation.
   If you seriously think there is no memory leak in your code
   but it fails with EXPECTED_DEPTH_TO_PASS,
   please manipulate it and report us the actual output.
   
   Orignally written by Godmar Back <godmar@gmail.com>
   Modified by Minkyu Jung, Jinyoung Oh <cs330_ta@casys.kaist.ac.kr>
*/
```
1. 자식이 fork가 실패할 때까지 **반복적으로 fork**한다. 최소 28개의 복사본이 run되는 것을 기대한다 
2. `exec()` 반환 값 : 만약 프로세스가 **실제로 시작되지 않았다면 `exec()`은 -1을 반환해야** 한다.(PID를 반환하는게 아님)
3. 1~2번 과정을 10번 반복 : kernel이 매번 같은 depth level을 허용하는지 체크하는 용도
4. 일부 프로세스는 자원 할당한 후 비정상적으로 종료되는 자식 프로세스를 생성할 것 

#### 1.2.2.  체크 포인트 
일단 핵심은 `exec()` 시스템 콜에서 오류가 나도 반환 값이 -1로 설정되어 있는지 확인해야 한다.
- [ ] ❓`exec()`반환 값 확인 


### 1.3.  핵심 로직 1. main 
```c
int main (int argc UNUSED, char *argv[] UNUSED) {
  test_name = "multi-oom";

  msg ("begin");

  int first_run_depth = make_children ();
  CHECK (first_run_depth >= EXPECTED_DEPTH_TO_PASS, "Spawned at least %d children.", EXPECTED_DEPTH_TO_PASS);

  for (int i = 0; i < EXPECTED_REPETITIONS; i++) {
    int current_run_depth = make_children();
    if (current_run_depth < first_run_depth) {
      fail ("should have forked at least %d times, but %d times forked", 
              first_run_depth, current_run_depth);
    }
  }

  msg ("success. Program forked %d iterations.", EXPECTED_REPETITIONS);
  msg ("end");
}
```
흠.. 일단 make_children()함수를 부르는 구나  


### 1.4.  핵심 로직 2. make_children

#### 1.4.1.  코드 및 분석 
```c
int make_children (void) {
  int i = 0;
  int pid;
  char child_name[128];
  for (; ; random_init (i), i++) {
    // i가 10/2 = 5 보다 클 때 
    if (i > EXPECTED_DEPTH_TO_PASS/2) {
      snprintf (child_name, sizeof child_name, "%s_%d_%s", "child", i, "X");
      pid = fork(child_name);
      if (pid > 0 && wait (pid) != -1) {
        fail ("crashed child should return -1.");
      } else if (pid == 0) {
        consume_some_resources_and_die();
        fail ("Unreachable");
      }
    }

		// 어떤 경우든 
    snprintf (child_name, sizeof child_name, "%s_%d_%s", "child", i, "O");
    pid = fork(child_name);
    if (pid < 0) {
      exit (i);
    } else if (pid == 0) {
      consume_some_resources();
    } else {
      break;
    }
  }

  int depth = wait (pid);
  if (depth < 0)
	  fail ("Should return > 0.");

  if (i == 0)
	  return depth;
  else
	  exit (depth);
}
```
for문을 보면 i가 5보다 큰 경우에 추가 분기에 걸리는 것을 볼 수 있다.
하지만 일단 이 경우는 차치하고 i가 어떤 숫자이든 상관없이 걸리는 로직을 집중적으로 보자 

```c
		// 어떤 경우든 
    snprintf (child_name, sizeof child_name, "%s_%d_%s", "child", i, "O");
    pid = fork(child_name);
    if (pid < 0) {
      exit (i);
    } else if (pid == 0) {
      consume_some_resources();
    } else {
      break;
    }
```
`child_name_i_O`라는 쓰레드 명을 가진 자식을 fork하는 것을 볼 수 있다. 
pintos에서 fork의 반환 값은 기본적으로 **부모의 경우 자식의 pid, 자식의 경우 0을 반환**한다. 
일단은 그게 잘 처리되어 있는지 확인을 해야 할 것이다.
- [x] ❓fork 반환 값 확인 

그 다음 볼 로직은 pid결과 값에 따라 분기처리 된 `if`, `else if`, `else` 문이다
1. `if (pid < 0)`
	- `pid`가 음수라는 말은 `fork`가 실패했다는 것이다.
	- 중요한 거는 exit 시스템 콜을 불러도 현재 프로세스의 부모 프로세스는 살아있어야 한다
	- [ ] ❓*현재 프로세스가 죽어도 부모 프로세스가 살아있도록 설계됐는지 확인* 💢
		- 아직 잘 모르겠다. 이게 해결되려면 `sys_exit`코드를 고쳐야 하는데.... 나중에 보자 
	  
2. `else if (pid == 0)` 
	- pid가 0이라는 말은 자식 프로세스일 경우이다.
	- 이 경우 `consume_some_resources()`라는 메서드가 처리되도록 간다.
	  
3. `else` 
	- pid > 0 보다 큰 경우이다. 이 경우는 부모 프로세스가 성공적으로 `fork` 후 for문을 탈출하는 것이다.
	- `for` 문 탈출 후 그럼 뭐하냐??
		- 바로 다음에 `int depth = wait(자식 pid)`가 있다. 즉 어차피 자식을 기다리기는 함 

#### 1.4.2.  체크 포인트 
- [x] ❓fork 반환 값 확인 
	- pintos에서 fork의 반환 값은 기본적으로 **부모의 경우 자식의 pid, 자식의 경우 0을 반환
	  
- [ ] ❓*현재 프로세스가 죽어도 부모 프로세스가 살아있도록 설계됐는지 확인* 💢
	- 아직 잘 모르겠다. 이게 해결되려면 `sys_exit`코드를 고쳐야 하는데.... 나중에 보자 



우선 if 문 3개를 본 뒤에 다음으로 확인할 것은 정해졌다. 바로 `consume_some_resources()`이다.

### 1.5.  핵심 로직 3. consume_some_resources()

#### 1.5.1.  코드 및 분석 
```c

/* Open a number of files (and fail to close them).
   The kernel must free any kernel resources associated
   with these file descriptors. */
static void consume_some_resources (void)
{
  int fd, fdmax = 126;

  /* Open as many files as we can, up to fdmax.
	 Depending on how file descriptors are allocated inside
	 the kernel, open() may fail if the kernel is low on memory.
	 A low-memory condition in open() should not lead to the
	 termination of the process.  */

  for (fd = 0; fd < fdmax; fd++) {
		if (open (test_name) == -1)
		  break;
  }
}
```

- **가능한 많은 파일들을 연다(fdmax인 126만큼) - 뭐야.. 이거밖에 없나?**
	- 근데 생각해보면 자식이 파일을 126번 연다고 가정하면 그 자식의 자식도 126번 여는거니까 fd가 정말 많아질 수 있긴 하네  
	  
- `Kernel` 내부에서 `file descriptor`가 어떻게 할당되는지에 따라, `open()`이 실패할 수도 있다(ex. 테이블 크기, 내부 자료구조, 메모리 부족 등)
- *`open()`실패 처리* 
	- **`open()`에서 low-memory(메모리 부족) 상황이 발생하더라도, 그 자체가 프로세스 종료로 이어지면 안 된다.** 
	- 즉, open실패가 단순히 open() 호출이 실패한 것으로 처리해야 한다는 말

목적
1. 커널이 많은 수의 파일을 열 수 있도록 허용하는지
2. 만약 자원 부족으로 커널 `open()`이 실패해도 그걸 정상적인 동작으로 간주하고 프로세스를 살리는지 

#### 1.5.2.  체크 포인트 
- [x] ❓*open 시 -1을 반환하도록 설정되었는가❓*
	- open 실패가 전체 프로세스의 종료로 이어지면 안된다
	  


일단 i의 모든 경우에서 진행되는 로직을 봤으니 다음은 `if (i > EXPECTED_DEPTH_TO_PASS/2)`의 경우를 볼 것이다.

### 1.6.  핵심 로직 3. make_children 내부 분기 로직 
이전에 make_children에서 `if (i > EXPECTED_DEPTH_TO_PASS/2)`인 경우를 넘어갔는데 이제 볼 것이다.

#### 1.6.1.  코드 및 분석 
```c
static const int EXPECTED_DEPTH_TO_PASS = 10;

int make_children (void) {
  int i = 0;
  int pid;
  char child_name[128];
  for (; ; random_init (i), i++) {
    if (i > EXPECTED_DEPTH_TO_PASS/2) {
      snprintf (child_name, sizeof child_name, "%s_%d_%s", "child", i, "X");
      pid = fork(child_name);
      if (pid > 0 && wait (pid) != -1) {
        fail ("crashed child should return -1.");
      } else if (pid == 0) {
        consume_some_resources_and_die();
        fail ("Unreachable");
      }
    }
```
i가 5보다 크면 진행되는 로직이다.
`fork`를 진행하는데 이전에 언급했던 `fork`반환 값 개념을 도입하면
- `if`문은 부모가 타는 분기이고, `else if` 는 자식이 타는 분기이다.
- 부모는 여기서 `wait()`을 또 호출하고 이 `wait`이 -1을 반환하면 `fail`로직이 가도록 설계 되어있다.
- 아래 else-if문을 안 봐도 유추할 수 있는 것은 `wait` 후 자식이 실패하는 것을 기대하고 자식이 실패할 경우 wait은 -1을 반환해야 하는 걸 유추할 수 있다.
- 따라서 이 부분이 먼저 처리됐는지 확인이 필요

#### 1.6.2.  체크 포인트 
- [ ] ❓부모가 자식 `wait` 후 깼을 때 자식의 성공/실패 상태에 따라 반환 값을 달리하는 가❓
	```c
	 tid_t sys_exec(char *command_line){
		// command_line은 유저 포인터니까 커널 포인터로 변경 (init에서 했던 것 처럼)
		if (validate_user_vaddr(command_line) == false){
			thread_current()->exit_status = -1;
			return TID_ERROR;
		}
		tid_t tid = syscall_process_execute(command_line);
		if (tid < 0){
			thread_current()->exit_status = -1;
			return TID_ERROR;
		} 
		return tid;
	 // ....
	int process_wait (tid_t child_tid UNUSED) {
			 int status = child->exit_status;
			 list_remove(&child->child_elem);
			 // 자식 스레드 깨우기
			 sema_up(&child->exit_sema);
			 return status;	 
	```
	
```c
tid_t process_fork (const char *name, struct intr_frame *if_ UNUSED) {
  /* Clone current thread to new thread.*/
  struct thread *parent = thread_current ();
  
  ....
	// 3. thead_create() 호출 (전달할 데이터 전달하기)
	tid_t tid = thread_create (name, PRI_DEFAULT, __do_fork, (void *) args);

	// 4. 자식 신호 대기
	sema_down(&parent->fork_sema);

	// 5. 깬 뒤 메모리 정리 
	palloc_free_page (args);

	//✅ 추가 : fork 했는데 자식이 뭔가 실패해서 자원 회수당한 경우 
	if (tid == TID_ERROR || find_child_thread_by_tid(tid) == NULL){
		return TID_ERROR;
	}  
```


### 1.7.  핵심 로직 4. consume_some_resources_and_die
근데 이 함수를 보기전에 이전 이 함수를 호출한 주체의 의도를 파악해야한다.
#### 1.7.1.  이 함수 호출한 주체의 의도
```c
  for (; ; random_init (i), i++) {
    printf("테스트 중 i = %d", i);
    if (i > EXPECTED_DEPTH_TO_PASS/2) {
      snprintf (child_name, sizeof child_name, "%s_%d_%s", "child", i, "X");
      pid = fork(child_name);
      if (pid > 0 && wait (pid) != -1) {
        fail ("crashed child should return -1.");
      } else if (pid == 0) {
        consume_some_resources_and_die(); // ⬅ 여기를 호출 
        fail ("Unreachable");
      }
```
`make_children`의 `else if` 문에서 이 함수가 호출됐었다.
결국 이 테스트가 통과할려면 fail에 도달하지 않는 것이 중요하다.
따라서, `consume_some_resources_and_die`호출 시 **이 함수 내부에서 종료를 시켜야 하고 그 종료 로직을 찾는 것**이 `consume_some_resources_and_die`를 풀어가는 과정의 일부 핵심일 것이라고 추측할 수 있다.


#### 1.7.2.  코드 및 분석 
```c
/* Consume some resources, then terminate this process
   in some abnormal way.  */
static int NO_INLINE
consume_some_resources_and_die (void)
{
  consume_some_resources ();
  int *KERN_BASE = (int *)0x8004000000;

  switch (random_ulong () % 5) {
	case 0:
	  *(int *) NULL = 42;
    break;

	case 1:
	  return *(int *) NULL;

	case 2:
	  return *KERN_BASE;

	case 3:
	  *KERN_BASE = 42;
    break;

	case 4:
	  open ((char *)KERN_BASE);
	  exit (-1);
    break;

	default:
	  NOT_REACHED ();
  }
  return 0;
}
```
- 랜덤으로 0 ~ 4의 case에 도달한다.
- 이 case들은 모두 정상적으로 보이지 않는다.

>[!tip] 일단 이 함수의 호출 의도에서 말했듯이 절대 return을 하게 하면 안된다.
>- 그러기 위해서 어떻게 해야할지 생각하는게 주요 과제처럼 보인다.
>- 이 목적을 이해하고 Case 5개를 어떻게 해결할 지 알아보자 


1. *case 0 - `*(int *) NULL = 42;`*
	- `NULL` 주소에 값을 쓰면 ➡ `PageFault`오류가 뜰 것 이 부분 처리가 일단 핵심. 
	- 이를 예외 처리로 잘 잡고 **해당 프로세스를 강제 종료하는 것**이 핵심 

2. *case 1 -  `*(int *) NULL;`*
	- `NULL`주소에서 읽기 ➡ 읽기에서 즉시 `Page Fault`
	- **읽기 처리가 필수** 
	  
3. *case 2 -  `*KERN_BASE`*
	- 커널 영역을 읽기 시도하는 것이다.
	- 유저는 커널 가상주소를 읽으면 안된다. 즉, 주소 검증이 제대로 이루어지고 프로세스 종료로 귀결되는지가 핵심 
	  
4. *case 3 - 	`*KERN_BASE = 42;`*
	- 커널 영역 쓰기 시도하는 것 
	- 커널 영역 쓰기를 절대 허용하지 않음. 유저 프로세스만 죽이고 커널은 안전해야 함 
	  
5. *case 4 - 	`open ((char *)KERN_BASE);  exit (-1);*
	- 커널 주소를 시스템 콜 인자로 전달하는 것
	- open 시스템 콜 진입 시 유저 point 검증인지 확인하는 것 
		- 올바르게 종료하면 `exit(-1)`까지 도달하지도 않음 


결국 이 함수의 목적은 User_Process가 잘못된 접근 or 잘못된 포인터를 넘겼을 때, **panic없이 해당 프로세스만 종료하고 자원을 회수하는지** 테스트하는 것이다.


null을 읽는다던가, NULL주소에 값을 쓴다던가 이런게 전부 시스템 콜 핸들러에게 가는 것이 아니라 **바로 `Page_Fault` 예외(트랩)을 발생시키고 `Kernel`의 예외 핸들러가 호출되는 것**이다. 따라서 이 부분을 다 처리하려면 `Page_fault` 부분을 수정하는 것이 중요. Pintos에서 구현된 page_fault는 아래와 같다.
```c

/* Page fault handler.  This is a skeleton that must be filled in
   to implement virtual memory.  Some solutions to project 2 may
   also require modifying this code.

   At entry, the address that faulted is in CR2 (Control Register
   2) and information about the fault, formatted as described in
   the PF_* macros in exception.h, is in F's error_code member.  The
   example code here shows how to parse that information.  You
   can find more information about both of these in the
   description of "Interrupt 14--Page Fault Exception (#PF)" in
   [IA32-v3a] section 5.15 "Exception and Interrupt Reference". */
static void
page_fault (struct intr_frame *f) {
	bool not_present;  /* True: not-present page, false: writing r/o page. */
	bool write;        /* True: access was write, false: access was read. */
	bool user;         /* True: access by user, false: access by kernel. */
	void *fault_addr;  /* Fault address. */

	/* Obtain faulting address, the virtual address that was
	   accessed to cause the fault.  It may point to code or to
	   data.  It is not necessarily the address of the instruction
	   that caused the fault (that's f->rip). */

	fault_addr = (void *) rcr2();

	/* Turn interrupts back on (they were only off so that we could
	   be assured of reading CR2 before it changed). */
	intr_enable ();

	/* Determine cause. */
	not_present = (f->error_code & PF_P) == 0;
	write = (f->error_code & PF_W) != 0;
	user = (f->error_code & PF_U) != 0;

	if (user) {
		sys_exit(-1);
		return;
	} 

#ifdef VM
	/* For project 3 and later. */
	if (vm_try_handle_fault (f, fault_addr, user, write, not_present))
		return;
#endif

	/* Count page faults. */
	page_fault_cnt++;

	/* If the fault is true fault, show info and exit. */
	printf ("Page fault at %p: %s error %s page in %s context.\n",
			fault_addr,
			not_present ? "not present" : "rights violation",
			write ? "writing" : "reading",
			user ? "user" : "kernel");
	kill (f);
}
```




```c
static void
page_fault (struct intr_frame *f) {
	bool not_present;  /* True: not-present page, false: writing r/o page. */
	bool write;        /* True: access was write, false: access was read. */
	bool user;         /* True: access by user, false: access by kernel. */
	void *fault_addr;    

	....
	
	// case 0, 1. NULL 접근 시 처리 
  if (fault_addr == (void *)NULL){
    sys_exit(-1);
    return;
  }

  // case 2, 3, 4. 유저가 커널 영역 읽기, 쓰기 금지 처리
  if (user && is_kern_pte((uint64_t *)fault_addr)) {
    sys_exit(-1);
    return;
  }
```


### 1.8.  중간 점검
얼추 위의 내용들을 체크하고 점검하고 수정하고 나니 아래와 같은 오류가 났다.
```c
Running multi-oom in batch mode... 
$ pintos -T 600 -k -v -m 20 -m 20 --fs-disk=10 -p tests/userprog/no-vm/multi-oom:multi-oom -- -q -f run 'multi-oom'

FAIL tests/userprog/no-vm/multi-oom
run: Should return > 0.: FAILED
FAIL
test 1/1 finish

=== Test Summary ===
Passed: 0
Failed: 1
  - multi-oom
    
    
///
  int depth = wait (pid);
  if (depth < 0)
    fail ("Should return > 0.");    
```
일단은 초반에 생겼던 `page_fault`오류 부분은 무사히 넘긴 것 같다. 왜냐하면 `consume_some_resources_and_die`부분에서 생겼던 오류가 아니라 그 다음에 진행되는 `if (depth < 0)`에서 오류가 생겼기 때문이다.

흠... `wait`가 반환값이 이상하다고??? 
일단 내가 구현한 로직에서 `exit_status`는 초기값이 0이고 오류 발생 시 -1로 설정해놓고 있다.


일단 테스트 코드에 임의로 `printf`를 넣어 핵심 변수값이 어떻게 돌아가고 있는지 확인하고자 했다. 
```c
현재 pid = 0
테스트 중 i = 9
Page fault at 0: not present error reading page in user context.
child_9_X: exit(-1)
현재 pid = 0
테스트 중 i = 10
Page fault at 0: not present error writing page in user context.
child_10_X: exit(-1)
Page fault at 0: not present error writing page in kernel context.
child_9_O: exit(-1)
depth 확인 전 pid 뭔데 : 17
depth 뭔데 : -1
(multi-oom) Should return > 0.: FAILED
child_8_O: exit(1)
depth 확인 전 pid 뭔데 : 15
depth 뭔데 : 1
```
- `page_fault` 에러처리는 제대로 되고 있고,
- **문제는 depth가 음수**라는 것이다. 그렇다면 wait가 문제일까?? 해서 봤는데 딱히 모르겠다. 근데 다시 생각해보면 wait에 넘기는 변수 값은 `pid`이고 그 값은 위의 출력값을 보아 0으로 계속 되는 것을 볼 수 있다.
- 그렇다면 pid는 어디서 초기화되는가? 바로 fork의 반환 값으로 초기화된다. `pid = fork(child_name);`


```c
  int depth = wait (pid);
  printf("depth 확인 전 pid 뭔데 : %d\n", pid);
  printf("depth 뭔데 : %d\n", depth); // -1 반환
  if (depth < 0)
    fail ("Should return > 0.");
```
현재 내 코드에서 fork값은 지금 


```text
만약 pid (자식 프로세스)가 exit() 함수를 호출하지 않고 커널에 의해서 종료된다면 
(e.g exception에 의해서 죽는 경우), wait(pid) 는 -1을 반환해야 합니다.
```


```
예외에서 잠들다child_7_X: exit(-1)
테스트 중 2 i = 7
테스트 중 2 i = 7
부모는 종료함
부모가 wait 전 pid = 13
현재 pid = 0
테스트 중 i = 8
Page fault at 0x8004000000: rights violation error writing page in user context.
예외에서 잠들다child_8_X: exit(-1)
테스트 중 2 i = 8
테스트 중 2 i = 8
부모는 종료함
부모가 wait 전 pid = 15
현재 pid = 0
테스트 중 i = 9
Page fault at 0x2: not present error reading page in kernel context.
예외에서 잠들다child_9_X: exit(-1)
테스트 중 2 i = 9
테스트 중 2 i = 9
부모는 종료함
부모가 wait 전 pid = 17
현재 pid = 0
테스트 중 i = 10
Page fault at 0: not present error writing page in user context.
예외에서 잠들다child_10_X: exit(-1)
테스트 중 2 i = 10
부모는 종료함
부모가 wait 전 pid = 19
```

갑자기 저 메시지를 보면서 든 생각은 부모가 `wait`를 하는데 자식의 자식의 자식~ 까지 기다린다는 것이다.
근데 FAQ GitBook의 `wait`부분을 보면 
- 자식들은 상속되지 않는다는 점을 알아두세요 :  만약 A 가 자식 B를 낳고 B가 자식 프로세스 C를  낳는다면, **A는 C를 기다릴 수 없습니다.** ⭐
- 위와 같은 조건이라면 `wait`은 즉시 fail 하고 -1을 반환합니다

즉, 부모가 손자까지 기다리는 상황이 안 나오도록 해야 하고 이런 상황이 나오려 하면 -1을 반환하고 FAIL해야한다는 뜻이다.

- 일단 `wait`부분이랑 어딜 고쳐야될지 추측해야 한다.
- 이를 추측하기 위해서 부모의 로직에서 `process_wait`하고 잠드는 부분을 breaking point걸어둔 다음에 그 다음에 어디가 실행되는지를 봤다. 
- `fork` 시스템 콜이 호출됐다. 
- 그렇다면 자식이 `fork`할 때 부모가 자식을 기다리고 있는지 확인하면 좋을 것 같다.
```c
// fork 로직에서 
tid_t process_fork (const char *name, struct intr_frame *if_ UNUSED) {
	....
  // 3. thead_create() 호출 (전달할 데이터 전달하기)

  tid_t tid = thread_create (name, PRI_DEFAULT, __do_fork, (void *) args);

  // 4. 자식 신호 대기
  sema_down(&parent->fork_sema);
```
나의 시나리오에서 이 로직을 실행시키는 놈은 자식이다. 즉, 자식이 `sema_down`으로 잠들어서 자식의 자식을 기다리기 전에 만약 이 놈의 부모가 기다리고 있으면 부모를 깨우고 기다리는 작업이 필요하다는 것이다.
아래와 같이 바꿔서 fork를 진행하려는 프로세스의 부모를 깨워주는게 좋지 않을까?
```c
tid_t process_fork (const char *name, struct intr_frame *if_ UNUSED) {
  
  // 3. thead_create() 호출 (전달할 데이터 전달하기)
  tid_t tid = thread_create (name, PRI_DEFAULT, __do_fork, (void *) args);

  // 현재 프로세스의 부모가 기다리고 있을 경우 깨워주기
  if (parent->parent && parent->is_waited){
    sema_up(&(parent->parent->exit_sema));
  }
```


```c
    if (i > EXPECTED_DEPTH_TO_PASS/2) {
      snprintf (child_name, sizeof child_name, "%s_%d_%s", "child", i, "X");
      printf("5이상인 i 전용 로직에서 fork 전 pid = %d\n", pid);
      pid = fork(child_name);
      printf("5이상인 i 전용 로직에서 fork 후 pid = %d\n", pid);
      if (pid > 0 && wait (pid) != -1) {

//현재 pid = 0
//테스트 중 i = 6
//5이상인 i 전용 로직에서 fork 전 pid = 0
//5이상인 i 전용 로직에서 fork 후 pid = 0      
```
어라?? pid가 0으로 초기화 되면서 부모는 


```c
int process_wait (tid_t child_tid UNUSED) {
  struct thread *parent = thread_current()->parent;
  if (parent && parent->is_waited){
    sema_up(&(parent->wait_sema));
  }
```


> [!WARNING] 자식이 사망했는데 부모가 무한히 기다리는 상황 발생 

```c

  printf("부모가 자식(pid = %d)을 기다립니다.\n", pid);
  int depth = wait (pid);
```


### 1.9.  wait 로직 초기화 
Gitbook을 잘못 이해하고 있었다.
부모가 자식의 자식까지 간접적으로 기다리는 것은 전파가 되게 해야했다.


따라서 다시 리셋 시켰다.

## 2.  트러블 슈팅으로 

[[Pintos/Pintos User Prog/Multi-OOM 테스트 Trouble(1) - 정상 흐름 속 page fault + 레지스터 오염\|Multi-OOM 테스트 Trouble(1) - 정상 흐름 속 page fault + 레지스터 오염]]
[[Pintos/Pintos User Prog/Multi-OOM 테스트 Trobule(2) - sema 교체\|Multi-OOM 테스트 Trobule(2) - sema 교체]]
