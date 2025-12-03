---
{"dg-publish":true,"permalink":"/Pintos/Pintos User Prog/Pintos U.P - Process_wait 구현해보기/","noteIcon":"","created":"2025-12-03T16:03:22.643+09:00","updated":"2025-12-03T16:09:29.391+09:00"}
---



## 1.  초기 Pintos 상태 
초기 구현된 것 
```c
/**
 * 기능
 * - 부모 스레드가 특정 프로세스의 종료를 기다리도록 하는 함수
 * - 즉, 부모가 특정 자식이 끝날 때까지 block하고 exit status를 회수하는 기능
 * 활용해야 할 것들 : 부모-자식 관계, 세마포어, 자식 상태를 나타내는 구조체 등을 이용해 동기화
 * 커널 안에서만 쓰인다
 *
 * vs System Call wait
 * - 시스템 콜 wait는 사용자 프로세스/스레드가 호출하는 인터페이스이다.
 * - 반면, process_wait는 커널 내부에서 실제 동작하는 함수
 */
int process_wait (tid_t child_tid UNUSED) {
  // while (1);
  return -1;
}
```
해야할 일
1. 매개 변수(child_tid)를 사용하여 자식 프로세스의 디스크립터 검색 
2. 자식 프로세스가 종료될 때까지 Block
3. 자식 종료 시 할당된 디스크립터 해제 

## 2.  어떻게 부모-자식 동기화에 wait이 사용❓
Preview : 간단 요약

1. 부모는 자식들을 추적하는 `child_status` 레코드를 갖는다.
2. 부모가 `process_wait(tid)` 호출 시 `tid`에 해당하는 1번 레코드를 찾아 `sema_down()`해서 `block`
3. 자식이 `exit()`, `process_exit()`시 여러 상태 바꾸고 `sema_up()`으로 부모를 깨운다
4. 부모는 깨어나서 상태 읽고 레코드 free -> status 반환 

## 3.  구현 
### 3.1.  구조체 설정 


```c
struct child_status {
	tid_t tid;
	int exit_status;
	bool is_exited;
	bool is_waited;
	struct semaphore wait_sema;
	struct list_elem elem;
};

struct thread {
	/* Owned by thread.c. */
	tid_t tid;                          /* Thread identifier. */
	....
	uint64_t *pml4; // 프로세스 공간 

	struct list children;  							/* 자식들의 child_status 목록 */
	struct child_status *self_satus;
	int exit_status;
```

`self_status`
- 자식이 종료할 때 상태를 입력해줘야 하는데 `self_status`에 적음
- self_satus에 적은 뒤 부모를 깨우고 부모는 상태를 보고 판단하는 것 
- 괜히 부모의 리스트 뒤지는 것보다 이게 나음 Cuz 락 경합, 혼동

`exit_stauts`
- 종료 여부를 저장하는 곳
- Why `child_status` 구조체 내에 있는데 `thread` 내에 또 기록하는가 ❓
	- 이유 = 수명분리 : 부모가 없어지거나 자식이 이미 정리된 경우 위험 
	- `thread`내의 `exit_status`는 자식(본인)의 상태를 언제든지 기록할 수 있다.
	- `child_status.exit_status` : 부모가 wait에서 읽고 해제할 결과 저장소 
	- 만약, 부모가 없거나 자식이 이미 정리된 경우 |



> [!WARNING] process_exec에 자식 생성/child_status 작업하지 않기
> process_exec은 단지 현재 스레드를 갈아끼우는 작업이다.


### 3.2.  구체 코드 

```c
int process_wait (tid_t child_tid UNUSED) {
	/**
	 * 1. 인자 tid에 해당하는 child thread 찾기 
	 * 2. 자식 스레드가 종료될 때까지 대기 - sema_down
	 * 3. 자식 스레드를 자식 리스트에서 제거 Cuz 자식 스레드 종료 예정
	 * 4. 자식 스레드 깨우기 - 자식도 정리하라고 
	 * 5. 자식 스레드의 종료 상태를 반환 
	 */
	struct thread *child = find_child_thread_by_tid(child_tid);
	if (child == NULL || child->is_waited){
		return -1;
	}
	
	// 부모 프로세스가 기다리고 있으면 깨워주기
	struct thread *cur = thread_current();
	struct thread *parent = cur->parent;
	if (parent && parent->is_waited) {
		sema_up(&cur->wait_sema);
		cur->exit_status = 0;
		sema_down(&cur->exit_sema);
	}

	child->is_waited = true;
	
	// 2. 자식 종료까지 대기 
	sema_down(&child->wait_sema);
	child->is_waited = false;

	int status = child->exit_status;
	if (status < 0) {
		list_remove(&child->child_elem);
	}
	// 자식 스레드 깨우기 
	sema_up(&child->exit_sema);
	return status;
}
```