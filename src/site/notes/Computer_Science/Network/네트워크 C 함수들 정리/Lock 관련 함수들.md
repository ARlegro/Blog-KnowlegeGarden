---
{"dg-publish":true,"permalink":"/Computer_Science/Network/네트워크 C 함수들 정리/Lock 관련 함수들/","noteIcon":"","created":"2025-12-03T14:52:46.070+09:00","updated":"2025-12-13T09:26:25.508+09:00"}
---


```c
// 초기화 / 해제
int pthread_rwlock_init(pthread_rwlock_t *rwlock, const pthread_rwlockattr_t *attr);
// int pthread_rwlock_destroy(pthread_rwlock_t *rwlock); 안 씀 

// 읽기 락
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);
// 쓰기 락
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);

// 락 해제
int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);
```

### `pthread_rwlock_init` - 읽기-쓰기 락 초기화
- *대략적 역할* : 멀티 스레드 환경에서 발생할 수 있는 **race condition을 방지하고자** 하는 초기화 메서드이다.
- *구체적 역할* : `pthread_rwlock` **객체를 초기화** 
	- `pthread_rwlock`의 내부 필드(락 모드, 대기 큐, 락 회수 등)를 초기화한다.
- *함수 내부 구현 구조* 
	```c
	/* Initialize read-write lock RWLOCK using attributes ATTR, or use
	   the default values if later is NULL.  */
	extern int pthread_rwlock_init (pthread_rwlock_t *__restrict __rwlock, // 초기화할 read-wrrte lock 객체의 주소 
	                    const pthread_rwlockattr_t *__restrict  // 초기화할 객체에 부여할 속성 (보통 NULL 씀)
	                    __attr) __THROW __nonnull ((1));
	```

>[!QUESTION] rwlock(Read-Write Lock) 객체란?
>- POSIX 스레드 라이브러리(`pthread`)에서 제공하는 동기화 객체
>- 스레드 간 공유 자원 보호에 사용되는 객체이다.
>- 이 객체는 2가지 잠금 모드를 가진다.
>	1. *읽기 락(공유락)*
>		- 자원을 **읽기 전용으로만** 사용하는 것.
>		- 쓰기가 불가능하다 ❗
>	2. *쓰기 락(배타락)*
>		- 읽기뿐만 아니라 쓰기도 못 하게 한다.
>		- **자원을 수정할 때** 이 모드를 사용한다 for 데이터 무결성

> [!info] rdlock(읽기 락) vs wrlock(쓰기 락)
> - `rdlock` : 여러 읽기 병행 허용✅, 쓰기는 금지❌
> - `wrlock` : 읽기/쓰기 모두 락❌

*✅pthread_rwlock_t의 내부 구조*
```c
// 
typedef union
{
  struct __pthread_rwlock_arch_t __data;
  char __size[__SIZEOF_PTHREAD_RWLOCK_T];
  long int __align;
} pthread_rwlock_t;
```

>[!QUESTION] 어떻게 pthread_rwlock_t로 특정 구조체를 락 걸까?
- 이 자료형에 Lock이 적용되면 Lock의 공유 자원 전체에 영향을 줌 
	```c
	typedef struct {
		cache_block_t *head; // LRU 기준: 가장 오래된 블록
		.... 
		pthread_rwlock_t lock; // 읽기/쓰기 보호용 락 ✔
	} cache_list_t;
	```
- `pthread_rwlock_t`는 `cache_list_t`와 같은 구조체의 멤버로 포함되어, **해당 구조체 "전체"의 무결성을 보호**하는 데 사용됨
