---
{"dg-publish":true,"permalink":"/Computer_Science/Network/C-Network/Cache find에서 생길만한 동시성 문제/","noteIcon":"","created":"2025-12-03T14:52:46.139+09:00","updated":"2025-12-13T18:25:26.786+09:00"}
---



참고
- 깃 허브 주소 : https://github.com/ARlegro/web-server-impl
- [[Computer_Science/Network/C-Network/Proxy Cache 구현\|Proxy Cache 구현]]
- [[Computer_Science/Network/C-Network/Proxylab 요구사항 정리\|Proxylab 요구사항 정리]]
- [[Computer_Science/Network/네트워크 C 함수들 정리/Lock 관련 함수들\|Lock 관련 함수들]]
- [[Computer_Science/Network/네트워크 C 함수들 정리/Posix 쓰레드 라이브러리 함수들\|Posix 쓰레드 라이브러리 함수들]]

### 1. 상황 
![Pasted image 20250904031003.png](/img/user/supporter/image/Pasted%20image%2020250904031003.png)
- Proxy 서버는 여러 클라이언트 요청을 동시에 처리하기 때문에, 다수의 쓰레드가 동시에 **캐시(cache)** 에 접근할 수 있다.  
- 이때 특정 URI에 해당하는 **객체(ex. 이미지, 동영상 등)가 캐시에 저장되어 있다면** 서버에 재요청하지 않고 즉시 **클라이언트에 응답할 수 있도록**한다.
- 캐시는 전역 구조체 `cache_list`로 관리된다

#### 메인 캐시 구조체
```c
typedef struct _cache_list {
  cache_block *head; // 
  cache_block *tail;
  int cached_total_size;
  pthread_rwlock_t lock;  // 이 속성으로 락 상태 관리 가능 
} cache_list;
  
static cache_list *cache; // 전역 캐시
```
> [!WARNING] 문제 
>  - 캐시는 전역 구조체(`cache_list`)로 관리되므로, 동시에 읽기/쓰기할 때 **경쟁 조건(race condition)** 이 발생할 수 있다. 
>  - 따라서,  **동시성 제어**가 반드시 必

---
### 2. 캐시 탐색 과정 
1. **Read-Lock 획득** 
	
>[!QUESTION] 왜 Read-Lock을 거는가?
 **캐시 탐색 과정에서 연결 리스트를 순회하는데**, 만약 이 시점에 다른 쓰레드가 삽입/이동/삭제 등을 수행
한다면 리스트 무결성이 깨진다. 즉, **리스트 무결성을 방지하고자** Read-Lock을 거는 것
 
2. 전역 캐시 구조체를 **순회하면서 데이터 찾기** 
3. **캐시 탐색 결과에 따른 분기 처리**
	- *🥊캐시 Miss 시* : `read-lock` 해제 후 곧바로 miss 플래그 반환 <br>![Pasted image 20250904022015.png](/img/user/supporter/image/Pasted%20image%2020250904022015.png)
		
	- **캐시 Hit 시 - 오늘의 Highlight**(뒤에서 ㄱ)

### 3. Cache Hit 시 크게 해야 할 일 2가지
1. 캐시된 메모리를 **어딘가에 복사하기** : 클라이언트에 보내줄 데이터
2. LRU 갱신 : 캐시된 메모리를 **구조체에서 맨 뒤로 보내기** 
	![Pasted image 20250904023142.png](/img/user/supporter/image/Pasted%20image%2020250904023142.png)


### 4. ❌잘못된 Cache Hit 처리 패턴
![Pasted image 20250904003351.png](/img/user/supporter/image/Pasted%20image%2020250904003351.png)
![Pasted image 20250904003520.png](/img/user/supporter/image/Pasted%20image%2020250904003520.png)

```c
// Return : cash hit(1), cash miss(0)
int cache_find(char *uri, char *object_buf, int *buf_size){
  .....
  pthread_rwlock_rdlock(&(cache->lock));  // 전역 캐시 읽기 락 
  cache_block *block = seq_search_cache(uri);  // 캐시된 데이터 찾기 by uri
	.....
  // ✅ 캐시 hit 상황
  // 락 전환 : 읽기 락 -> 쓰기 락(구조체 내부 변경 필요하므로)
  pthread_rwlock_unlock(&(cache->lock));
  pthread_rwlock_wrlock(&(cache->lock));

  memcpy(temp, block->buf, block->size); // 캐시된 데이터 buf에 복사
  *buf_size = block->size;
  
  move_to_tail(block); // 구조체의 맨 뒤로 보내기 for LRU 알고리즘
  pthread_rwlock_unlock(&(cache->lock)); // 쓰기락으로 변경하기 전 unlock
  return 1;
```

![Pasted image 20250904004553.png](/img/user/supporter/image/Pasted%20image%2020250904004553.png)

> 여기서 문제 : **Lock 전환 과정에서 Race Condition이 발생** 🥊

![Pasted image 20250904015821.png](/img/user/supporter/image/Pasted%20image%2020250904015821.png)
*순서*
1. Read Lock을 건 상태에서 **캐시 데이터(A 블록)를 찾고**
2. **Lock수준 강화를 위해 UnLocking** - pthread_rwlock은 바로 전환 불가
3. **Write Lock으로 변경** 
4. 캐시 데이터(A 블록)를 **Client에게 전달할 buf에 복사** + `move_to_tail()`

올바른 갱신 

### 어떻게 올바르게 Cache-Hit 처리를 할까??
*💢앞선 방식의 문제*
- 캐쉬된 데이터를 찾았음에도 `unlock - write Lock` 사이에 `evict()`로직으로 인해 찾은 데이터가 사라지는 문제.
- 이렇게 되면, 클라이언트에 보낼 캐시 데이터가 사라지게 되어 문제 발생

*💚해결 방안*

![Pasted image 20250904025008.png](/img/user/supporter/image/Pasted%20image%2020250904025008.png)
*Cache Hit 후 흐름(읽기 락 상태)* 
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
  // 위의 search 한 block 안 쓰고 재조회 이유 : block이 이미 해제되었거나 다른 객체로 할당된 주소 일 수 있으므로
  cache_block *new = seq_search_cache(uri);
  if (new && new->size >0){
    move_to_tail(new); // LRU 알고리즘에 반영하기 위해 
  }
  pthread_rwlock_unlock(&(cache->lock));
  memcpy(object_buf, temp, *buf_size);
  return 1;
}
```