---
{"dg-publish":true,"permalink":"/Computer_Science/Network/C-Network/Proxy Cache 구현/","noteIcon":"","created":"2025-12-03T14:52:46.105+09:00","updated":"2025-12-09T17:19:41.882+09:00"}
---



참고 
- [[Computer_Science/Network/C-Network/Proxylab 요구사항 정리\|Proxylab 요구사항 정리]] 
- [[Computer_Science/Network/C-Network/Cache find에서 생길만한 동시성 문제\|Cache find에서 생길만한 동시성 문제]]
- 깃 허브 주소 : https://github.com/ARlegro/web-server-impl1

## 캐시에 사용할 구조체 보기 

### 1. 캐시 블록 구조 
필요한 것 정리
1. URI : 해당 cache 블록이 맞는지 key역할하는 
2. 실제 데이터 포인터 (heap에 할당된)
3. object 크기 
4. 이전/다음 블록 

```c
  

typedef struct _cache_block {
  char *uri; // 나중에 동적 할당하기
  char *buf; // 나중에 동적 할당하기
  int size; // buf 크기
  
  struct _cache_block *next;
  struct _cache_block *prev;

} cache_block;
```


### 2. 캐시 리스트 구조
필요한 것 정리
1. head - tail 캐시 블록
2. 현재 캐시된 전체 바이트 수
3. 읽기/쓰기 보호용 락 


```c
typedef struct _cache_list {
  cache_block *head;
  cache_block *tail;
  int cached_total_size;
  pthread_rwlock_t lock;
} cache_list;

static cache_list *cache; // 전역 캐시
```

## 캐시 관련 메서드 

### 1. init()
```c
// 전역 캐시 init

void cache_init(){
  cache = (cache_list *)Malloc(sizeof(cache_list)); // 전역 캐시 구조체 동적 할당 
  cache->head = NULL; // 캐시 첫 블록 초기화 
  cache->tail = NULL; // 캐시 마지막 블록 초기화 
  cache->cached_total_size = 0; // 메인 캐시에 저장된 캐시들 총 합 
  pthread_rwlock_init(&(cache->lock), NULL); // 읽기-쓰기 락 초기화
}
```

Proxy 서버가 시작되면 캐시를 위해 전역 캐시를 초기화한다.
`pthread_rwlock_init`를 제외하고는 정글 5,6,7 주차에서 했던 double_list, malloc에서 충분했으니 `pthread_rwlock_init`만 설명하려고 한다<br>

*`pthread_rwlock_init` - 읽기-쓰기 락 초기화*
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


### 2. cache_block_init - 캐시 블록 초기화 

> 캐시에 담을 블록을 초기화한다
> - 한번에 담는 것이 아니라 BUF형식으로 담을 시 미리 초기화 必

```C
cache_block *cache_block_init(char *uri){
  cache_block *block = (cache_block *)Malloc(sizeof(cache_block));
  block->buf = (char *)Malloc(MAX_OBJECT_SIZE);  // pdf 요구사항에서 명시했던 사이즈로 제한 
  block->uri = (char *)Malloc(strlen(uri) + 1); // '\0'도 포함시키기 
  strcpy(block->uri, uri);
  block->size = 0;
  block->next = NULL;
  block->prev = NULL;
  return block;
}
```


### 3. cache_evict - 캐시 
> - 캐시 용량 초과 시 데이터 저장할 공간 확보를 위해 오래된 블록을 제거하는 로직
> - LRU 캐시 정책을 구현하는 함수 

```c
// 캐시 용량 초과 시 오래된 블록을 제거하는 로직
void cache_evict(const int need_size){
	// 용량 초과 검사하면서 용량 될때까지 while문 
  while ((cache->cached_total_size + need_size) > MAX_CACHE_SIZE) 
  {
    cache_block *old = cache->head; // 가장 오래된 블록 선택
    cache->cached_total_size -= old->size; // 전체 캐시 용량 차감
    cache->head = old->next; // head 변경
    if (cache->head != NULL) {
      cache->head->prev = NULL;
    } else {
      cache->tail = NULL;
    }
    // 캐시에서 제거할 블록 + 내부 속성 Free
    Free(old->buf);
    Free(old->uri);
    Free(old);
  }
}
```
동작 흐름
1. **용량 초과 검사**
    - `need_size` 만큼 새 데이터를 넣었을 때 총 크기가 `MAX_CACHE_SIZE`를 넘는지 확인.
    - 넘으면 오래된 블록을 계속 제거해야 함.
        
2. **오래된 블록 선택 (head)**
    - 캐시는 **LRU(Least Recently Used)** 방식으로 관리 (PDF 요구사항)
    - 가장 오래된 블록은 리스트의 `head`에 위치.
        
3. **용량 갱신**
    - `cached_total_size`에서 해당 블록 크기를 빼줌.
        
4. **리스트 연결 갱신**
    - `head`를 다음 노드로 옮기고,        
    - 만약 더 이상 노드가 없으면(`head == NULL`) → `tail`도 `NULL`로 세팅.
        
5. **메모리 해제**
    - 블록이 가지고 있던 데이터(`buf`), URI 문자열(`uri`), 블록 구조체(`old`)까지 해제.


### 4. cache_insert 
> 캐시에 새로운 블록을 추가하는 함수

```C
void cache_insert(cache_block *block){
  if (cache == NULL || block == NULL) return;
  
  // 1. 캐시 용량 부족 시 cache_evit 必
  if (cache->cached_total_size + block->size > MAX_CACHE_SIZE){
    cache_evict(block->size);
  }

  // 2. 전역 캐시가 빈 리스트인 경우 : 첫번째 노드 삽입 
  if (cache->head == NULL) {
    cache->head = cache->tail = block;
    block->prev = block->next = NULL;
  }
  // 3. 전역 캐시가 빈 리스트가 아닌 경우
  else {
    cache->tail->next = block;
    block->prev = cache->tail;
    block->next = NULL;
    cache->tail = block;
  }
  cache->cached_total_size += block->size;  // 캐시 사이즈 업데이트
}
```

이중 리스트를 다루는 구체적인 설명은 이번 페이지에서는 생략하겠다(이중 리스트 주차때 많이 했음) <br>
*동작 흐름*
1. **용량 확보** by 크기 검사 및 `cache_evict()`
2. **이중 리스트 삽입**

### 5. seq_search_cache(uri)
> uri를 사용해서 전역 캐시를 탐색 
```c
cache_block *seq_search_cache(const char *uri){
  for (cache_block *t = cache->head; t != NULL; t = t->next){
    if (strcmp(t->uri, uri) == 0) return t; // uri 키를 가진 블록 찾기
  }
  return NULL;
}
```

### 6. move_to_tail
> **역할 : LRU 알고리즘으로 캐시 구조체를 관리해야 할 때 사용하는 함수** 

PDF 요구사항에는 다음과 같은 2가지 경우에 최근 Cache Block으로 간주한다.
1. **쓴 block**
2. ⭐**읽은 block** - 캐시를 읽어서 사용하기만 해도 최근 Block으로 간주 
```c
// LRU 알고리즘 시 최근 쓴거 뒤로 가게하는 것
void move_to_tail(cache_block *block){
  if (block == cache->tail) return;
  
  if (block == cache->head) {
    if ((cache->head = cache->head->next) != NULL) {
      cache->head->prev = NULL;
    }
  } 
  
  else {
    cache_block *prev = block->prev;
    cache_block *next = block->next;
    if (prev != NULL) prev->next = next;
    if (next != NULL) next->prev = prev;
  }
  
  // 꼬리로 옮기기
  cache->tail->next = block;
  block->prev = cache->tail;
  block->next = NULL;
  cache->tail = block;
}
```

구체적 로직은 이중 리스트 관련된거라 생략 

### 7. cache_find() ⭐
> 전역 캐시에서 찾는 Cache Block이 있는지 확인하는 메서드 ⭐

자세한 로직 및 주의할점 : [[Computer_Science/Network/C-Network/Cache find에서 생길만한 동시성 문제\|Cache find에서 생길만한 동시성 문제]]

```c
int cache_find(char *uri, char *object_buf, int *buf_size){

  char temp[MAX_OBJECT_SIZE];

  // 전역 캐시 읽기 락 Cuz 읽는 동안 삽입/이동/evict 시 리스트가 찢어짐
  pthread_rwlock_rdlock(&(cache->lock));
  cache_block *block = seq_search_cache(uri);
  // 1. 캐시 Miss 상황
  if (block == NULL || block->size <= 0){
    pthread_rwlock_unlock(&(cache->lock));
    return 0;
  }

  // 2. 캐시 hit 상황
  // 일단 메모리 
  memcpy(temp, block->buf, block->size);
  *buf_size = block->size;
  pthread_rwlock_unlock(&(cache->lock)); // 쓰기락으로 변경하기 전 unlock
  
  // 쓰기락으로 변경 후 재조회
  pthread_rwlock_wrlock(&(cache->lock));
  // 위의 search 한 block 안 쓰고 재조회 Cuz block이 이미 해제되었거나 다른 객체로 할당된 주소 일 수 있으므로
  cache_block *new = seq_search_cache(uri);
  if (new && new->size >0){
    move_to_tail(new); // LRU 알고리즘에 반영하기 위해 
  }
  pthread_rwlock_unlock(&(cache->lock));
  memcpy(object_buf, temp, *buf_size);
  return 1;
}
```
*Cache Hit 후 흐름(읽기 락 상태)* 
![Pasted image 20250904025008.png](/img/user/supporter/image/Pasted%20image%2020250904025008.png)
1. **Unlock 전에 미리 데이터 복사** 
	- 이렇게 하면 복사 데이터는 **일관성** 있음 
	- 복사한 **원본의 블록이 해제가 되더라도 복사본의 일관성** 유지  
	  
2. **읽기 락 해제**
3. **LRU 갱신을 위한 쓰기 락 획득**
4. **캐시 블락 재조회**
5. **move_to_tail**
6. **쓰기 락 해제** 
	  
>[!QUESTION] 왜 기존에 찾은 block 캐시를 안 쓰고 재조회?
> - Lock 전환 동안 block이 이미 해제되었거나 다른 객체로 할당된 주소 일 수 있으므로
> - 물론 재조회 시 block이 사라졌을 수 있지만,그건 **LRU 갱신 실패일 뿐, 클라이언트에 보낼 데이터(temp)는 이미 안전**.(**데이터 무결성이** LRU 일관성보다 **중요**)


### 언제 캐시 로직이 발동되나???

Client -> Proxy 로 진행된 뒤 end-server 소켓을 열기전에 Proxy에 캐시가 있는지 확인한다.

```c
void doit(int fd) {
  .... 
  
  // 캐시 키 생성 (hostname + path)
  snprintf(cache_key, MAXLINE, "%s%s", dest_host, dest_path);

  // ✅ 캐시 확인 befor end-server 연결 
  int is_cached = cache_find(cache_key, object_buf, &object_size);
  if (is_cached){
    Rio_writen(fd, object_buf, object_size);
    return;
  }

  change_request_headers(&rio, dest_host, header);  

  // server 소켓 연결하기
  int server_fd = Open_clientfd(dest_host, dest_port);
```
