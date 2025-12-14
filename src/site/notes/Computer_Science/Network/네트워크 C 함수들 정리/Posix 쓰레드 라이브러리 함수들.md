---
{"dg-publish":true,"permalink":"/Computer_Science/Network/네트워크 C 함수들 정리/Posix 쓰레드 라이브러리 함수들/","noteIcon":"","created":"2025-12-03T14:52:46.040+09:00","updated":"2025-12-13T18:25:26.914+09:00"}
---


> 유닉스 계열 POSIX 시스템에서 병렬 프로그래밍을 위한 C 언어 thread 라이브러리 
```c
#include <pthread.h>
```


>[!tip] fork()와 비교
>- `fork()`는 부모 프로세스의 리소스를 자식 프로세스가 '복사'해서 사용 
>- 반면, thread 방식은 부모의 소스, 시그널 핸들러, open file 객체 등을 **공유** 

---
### 1. 쓰레드에서 실행될 함수 정의 
```c
void *therad_fuction(void * arg){
	....

	return NULL;
}
```

*특징* 
- *Param* : `void *` 타입의 인자를 받는다
- *Return* : `void *` 타입

---
### 2. 쓰레드 생성 - pthread_create()

> [!WARNING] 우선, **생성될 쓰레드 ID를 저장할 `pthread_t` 타입 포인터 선언**

```c
/* Create a new thread, starting with execution of START-ROUTINE
   getting passed ARG.  Creation attributed come from ATTR.  The new
   handle is stored in *NEWTHREAD.  */
extern int pthread_create (pthread_t *__restrict __newthread, // 생성된 쓰레드의 ID 저장할 포인터
                  const pthread_attr_t *__restrict __attr,  // 쓰레드 속성 
                  void *(*__start_routine) (void *),  // 쓰레드가 실행할 함수 
                  void *__restrict __arg) __THROWNL __nonnull ((1, 3)); // 3번 함수로 넘길 인자 
```
*인자 분석*
- *첫 번째* : 생성될 쓰레드 ID를 저장할 포인터 
- *두 번째* : 쓰레드의 속성을 지정하는 포인터 (일반적으로 NULL)
- *세 번째* : 쓰레드가 실행할 함수
- *네 번째* : 3번 실행할 함수로 넘길 인자 

*반환 값* - error가 아닌 Return값으로 오류 확인 必
- *성공* : 0
- *실패* : 오류코드(양수)

✅ 간단 예제
```C
#include <pthread.h>
#include <stdio.h>

void *start_thread(void *arg);
  
int main(void){

  pthread_t pid1, pid2; // 생성될 쓰레드의 id 담는 변수
  // 1번째 쓰레드 생성
  int result1 = pthread_create(&pid1, NULL, start_thread, "thread1");
  if (result1){
    printf("thread1에서 에러 발생");
    return 1;
  }
	// 2번째 쓰레드 생성
  int result2 = pthread_create(&pid1, NULL, start_thread, "thread2");
  if (result2){
    printf("thread2에서 에러 발생");
    return 1;
  }
	....
  return 0;
}
  
void *start_thread(void *arg){
  const char* name = arg;
  printf("%s 쓰레드 시작합니다\n", name);
  return NULL;
}
```

---
### 3. pthread_join()

>  `thread`로 지정된 스레드가 종료할 때까지 **현재 스레드를 일시 중지**시키고, **지정한 스레드의 결과를 받아온다**

---
#### 효과 
1. *현재 스레드를 일시 중지*시킨다.
2. *종료된 스레드의 반환값*을 가져온다
3. *자원 반환 트리거 역할* 
	- 💢*기존 문제 - 쓰레드는 자동 반환이 아니다*
		- 쓰레드 **실행 흐름이 끝나도 스레드 자원(스택, TCB 등)은 대기 상태**로 남는다 (물론, 프로세스가 종료되면 해제되긴 함)
		- 이렇게 대기 상태에 빠지면 zombie thread형태로 남음
		  
	- ✅*해결*
		- join()을 호출하면 1,2번 효과 말고도 커널이 보관중이던 **쓰레드 자원을 실제로 해체하는 작업**을 한다


> [!WARNING] 한 쓰레드당 한 번만 호출할 수 있다.
> - 이유 : 첫 join()에서 이미 쓰레드의 자원을 회수했기 때문 
> - 만약, 한 번더 호출하면 메모리 쓰레기값을 얻거나 segment fault가 날 수 있다.

---
#### 함수 분석 
```c
/* Make calling thread wait for termination of the thread TH.  
	The exit status of the thread is stored in *THREAD_RETURN, if THREAD_RETURN is not NULL.
  This function is a cancellation point and therefore not marked with __THROW.  */
extern int pthread_join (pthread_t __th, void **__thread_return);
```
*인자 분석 *
- *첫 번째* : 대기할 스레드의 id 
- *두 번째* : 종료하는 스레드가 **반환하는 값을 저장할 포인터** (❗`void**` 타입)
	- `NULL`이면 : 
	- `NULL`이 아니면 : 

join()을 사용하지 않으면 메인 쓰레드는 특정 스레드의 작업을 기다리지 않고 

---
#### main 쓰레드가 기다리는 케이스
```c
int main(void){

  printf("[main 쓰레드 시작]\n");
  pthread_t pid1, pid2; // 생성될 쓰레드의 id 담는 변수
  int result1 = pthread_create(&pid1, NULL, start_thread, "thread1");
  int result2 = pthread_create(&pid2, NULL, start_thread, "thread2");
  pthread_join(pid1, NULL);
  pthread_join(pid2, NULL);
  printf("[main 쓰레드 종료]\n");
  return 0;
}

void *start_thread(void *arg){
  const char* name = arg;
  printf("%s 쓰레드 시작합니다\n", name);
  sleep(3);
  printf("%s 쓰레드 종료합니다\n", name);
  return NULL;
}

// 출력값 
// [main 쓰레드 시작]
// thread1 쓰레드 시작합니다
// thread2 쓰레드 시작합니다
// thread1 쓰레드 종료합니다
// thread2 쓰레드 종료합니다
// [main 쓰레드 종료]  <<<< 다 기다려줌
```
- 출력값을 보면 **main쓰레드가** thread1,2를 **전부 기다려주고 종료**했다.
- 이유 : **join을 사용해서** main가 thread1, thread1 끝나기를 기다리게 만들었기 때문이다.

---
#### 😢main 쓰레드가 기다리지 않는 case
```c
int main(void){

  printf("[main 쓰레드 시작]\n");
  pthread_t pid1, pid2; // 생성될 쓰레드의 id 담는 변수
  int result1 = pthread_create(&pid1, NULL, start_thread, "thread1");
  int result2 = pthread_create(&pid2, NULL, start_thread, "thread2");
  // pthread_join(pid1, NULL);
  // pthread_join(pid2, NULL);
  printf("[main 쓰레드 종료]\n");
  return 0;
}

void *start_thread(void *arg){
  const char* name = arg;
  printf("%s 쓰레드 시작합니다\n", name);
  sleep(3);
  printf("%s 쓰레드 종료합니다\n", name);
  return NULL;
}



// [main 쓰레드 시작]
// thread1 쓰레드 시작합니다
// thread2 쓰레드 시작합니다
// [main 쓰레드 종료]
```
> main 쓰레드가 thread1, 2가 끝나는 것을 기다려주지 않았다 (각 쓰레드의 종료 출력이 안됨)

>[!QUESTION] main 종료되면 다른 프로세스 자원은 어케돌까?
>- main이 종료된다 = 프로세스가 종료된다
>- 프로세스가 종료되면 거기에 속한 **모든 스레드가 즉시 강제종료**된다
>- **프로세스가 죽으면 OS가 알아서 자원들을 회수**하는데, 프로세스의 자원을 사용중인 쓰레드의 자원도 정리된다고 보면 됨 

---
### 4. pthread_detach()

> 실행 중인 쓰레드를 **분리(detach)상태로 만드는** 함수 

*효과*
1. *자원 자동 해제* : 특정 스레드의 자원을 자동으로 해제하도록 설정 (join으로 자원 회수가 필요 없음)

모든 thread는 기본적으로 joinable상태이다(`join()`할 수 있는). 근데 만약 이 thread를 join하지 못하게 하려면 `detach()`를 사용하면 된다.
```c
int main(void){
  printf("[main 쓰레드 시작]\n");
  pthread_t pid1; // 생성될 쓰레드의 id 담는 변수
  int result1 = pthread_create(&pid1, NULL, start_thread, "thread1");
  pthread_detach(pid1);  // ✅ detach로 join못하게
  pthread_join(pid1, NULL); // 🥊 이거 해도 detach 모드면 아무 효과 없음
  printf("[main 쓰레드 종료]\n");
  return 0;
}

void *start_thread(void *arg){
  const char* name = arg;
  printf("%s 쓰레드 시작합니다\n", name);
  sleep(3);
  printf("%s 쓰레드 종료합니다\n", name);
  return NULL;
}

// 출력값
// [main 쓰레드 시작]
// thread1 쓰레드 시작합니다
// thread2 쓰레드 시작합니다
// [main 쓰레드 종료]
```

>[!tip] 쓰레드 생성시에 detach 속성 설정 가능 
>처음부터 detach 모드 쓰레드로 만들 수 있다.
```c
  pthread_attr_t attr;
  // 1. thread 속성 객체 초기화
  pthread_attr_init(&attr);
  // 2. attr을 detach모드로 설정
  pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
  // 3. 쓰레드 생성 시 속성 적용
  pthread_create(&pid1, &attr, start_thread, "thread1");****
```

---
### 5. pthread_exit()

> 특정 스레드를 종료시킴 - 자원도 반환 시킴 

```c
/* Terminate calling thread.

   The registered cleanup handlers are called via exception handling
   so we cannot mark this function with __THROW.*/
extern void pthread_exit (void *__retval) __attribute__ ((__noreturn__));
```

---
#### main 스레드만 종료 시키기
다른 스레드 실행시키는 경우 

```c
int main(void){
  printf("[main 쓰레드 시작]\n");
  pthread_t pid1, pid2; // 생성될 쓰레드의 id 담는 변수
  int result1 = pthread_create(&pid1, NULL, start_thread, "thread1");
  int result2 = pthread_create(&pid2, NULL, start_thread, "thread2");
  printf("[main 쓰레드 종료]\n")
  pthread_exit(NULL);
  return 0;
}

void *start_thread(void *arg){
  const char* name = arg;
  printf("%s 쓰레드 시작합니다\n", name);
  sleep(3);

  printf("%s 쓰레드 종료합니다\n", name);
  return NULL;
}
// 출력값
// [main 쓰레드 시작]
// thread1 쓰레드 시작합니다
// thread2 쓰레드 시작합니다
// [main 쓰레드 종료]
// thread1 쓰레드 종료합니다
// thread2 쓰레드 종료합니다
```
- main쓰레드가 종료되고 **프로세스가 다른 쓰레드의 실행을 기다렸다.**
- *❌pthread_exit이 없었을 때❌*
	- main쓰레드가 return까지 도달하면서 프로세스가 종료됐었다.
	- 이 때는 다른 쓰레드를 기다려주지 않았음 

---
#### 쓰레드 함수에서 적용해보기 
```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

int main(void){
  printf("[main 쓰레드 시작]\n");
  pthread_t pid1, pid2; // 생성될 쓰레드의 id 담는 변수
  int result1 = pthread_create(&pid1, NULL, start_thread, "thread1");
  int result2 = pthread_create(&pid2, NULL, start_thread, "thread2");
  int *thread_result1;
  int *thread_result2;

  pthread_join(pid1, (void **)&thread_result1);
  pthread_join(pid2, (void **)&thread_result2);
  
  printf("thread_result1 = %d\n", *thread_result1);
  printf("thread_result2 = %d\n", *thread_result2)
  printf("[main 쓰레드 종료]\n");
  pthread_exit(NULL);
  return 0;
}

void *start_thread(void *arg){
  const char* name = arg;
  printf("%s 쓰레드 시작합니다\n", name);
  sleep(3);
  int *ret = malloc(sizeof(int));
  *ret = 100;
  printf("%s 쓰레드 종료합니다\n", name);
  pthread_exit((void *)ret);
}

// [main 쓰레드 시작]
// thread1 쓰레드 시작합니다
// thread2 쓰레드 시작합니다
// thread1 쓰레드 종료합니다
// thread2 쓰레드 종료합니다
// thread_result1 = 100
// thread_result2 = 100
// [main 쓰레드 종료]
9```
