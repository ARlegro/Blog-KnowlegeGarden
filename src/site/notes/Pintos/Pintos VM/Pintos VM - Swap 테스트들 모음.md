---
{"dg-publish":true,"permalink":"/Pintos/Pintos VM/Pintos VM - Swap 테스트들 모음/","noteIcon":"","created":"2025-12-03T14:52:52.672+09:00","updated":"2025-12-13T18:25:31.655+09:00"}
---



관련 개념 
- [[Pintos/Pintos VM/Pintos VM - Swapping\|Pintos VM - Swapping]]


### 0.1.  목차

- [[#1.  Swap-anon|1.  Swap-anon]]
	- [[#1.  Swap-anon#1.1.  테스트 코드 및 처음 테스트 결과|1.1.  테스트 코드 및 처음 테스트 결과]]
	- [[#1.  Swap-anon#1.2.  문제 지점 발견|1.2.  문제 지점 발견]]
	- [[#1.  Swap-anon#1.3.  수정 및 결과|1.3.  수정 및 결과]]
	- [[#1.  Swap-anon#1.4.  2차 문제|1.4.  2차 문제]]
	- [[#1.  Swap-anon#1.5.  문제 발견|1.5.  문제 발견]]
- [[#2.  swap-file|2.  swap-file]]
	- [[#2.  swap-file#2.1.  원인|2.1.  원인]]
	- [[#2.  swap-file#2.2.  문제 해결|2.2.  문제 해결]]


## 1.  Swap-anon

### 1.1.  테스트 코드 및 처음 테스트 결과 

```c
#define PAGE_SHIFT 12
#define PAGE_SIZE (1 << PAGE_SHIFT)
#define ONE_MB (1 << 20)  // 1MB
#define CHUNK_SIZE (20 * ONE_MB)
#define PAGE_COUNT (CHUNK_SIZE / PAGE_SIZE)  // 약 5120번 정도 

static char big_chunks[CHUNK_SIZE];

void test_main(void) {
  size_t i;
  void *pa;
  char *mem;

  for (i = 0; i < PAGE_COUNT; i++) {
	  // sparse = 희소한 
    if (!(i % 512)) msg("write sparsely over page %zu", i);
    mem = (big_chunks + (i * PAGE_SIZE));
    *mem = (char)i;
  }

  for (i = 0; i < PAGE_COUNT; i++) {
    mem = (big_chunks + (i * PAGE_SIZE));
    if ((char)i != *mem) {
      fail("data is inconsistent");
    }
    if (!(i % 512)) msg("check consistency in page %zu", i);
  }
}

```
- *테스트 목적* : 스왑이 일어나도 데이터 손실 없이 동일한 값이 복원되는지 
	- 매 페이지마다 작게 접근(1byte만)하는 희소 접근 패턴에서 lazy alloc, swap, consistency 테스트 하는 것 
	  
- *첫 for문* 
	- 한 페이지의 첫 byte만 접근한다.
	- 20MB의 배열을 연속적으로 접근하는 것이 아니라 512개의 페이지마다 접근하여 PF를 유발 

- *두번째 for문 - 일관성 검증* 
	- 첫 for문에서 쓴 값이 두 번째 for문에서 그대로 읽히는지 테스트
	- 페이지 교체 or Swap해도 데이터가 손상되지 않고 복원되는지를 테스트 

```c
Acceptable output:
  (swap-anon) begin
  (swap-anon) write sparsely over page 0
  (swap-anon) write sparsely over page 512
  (swap-anon) write sparsely over page 1024
  (swap-anon) write sparsely over page 1536
  (swap-anon) write sparsely over page 2048
  (swap-anon) write sparsely over page 2560
  (swap-anon) write sparsely over page 3072
  (swap-anon) write sparsely over page 3584
  (swap-anon) write sparsely over page 4096
  (swap-anon) write sparsely over page 4608
  (swap-anon) check consistency in page 0
  (swap-anon) check consistency in page 512
  (swap-anon) check consistency in page 1024
  (swap-anon) check consistency in page 1536
  (swap-anon) check consistency in page 2048
  (swap-anon) check consistency in page 2560
  (swap-anon) check consistency in page 3072
  (swap-anon) check consistency in page 3584
  (swap-anon) check consistency in page 4096
  (swap-anon) check consistency in page 4608
  (swap-anon) end
Differences in `diff -u' format:
  (swap-anon) begin
  (swap-anon) write sparsely over page 0
  (swap-anon) write sparsely over page 512
  (swap-anon) write sparsely over page 1024
- (swap-anon) write sparsely over page 1536
- (swap-anon) write sparsely over page 2048
- (swap-anon) write sparsely over page 2560
- (swap-anon) write sparsely over page 3072
- (swap-anon) write sparsely over page 3584
- (swap-anon) write sparsely over page 4096
- (swap-anon) write sparsely over page 4608
- (swap-anon) check consistency in page 0
- (swap-anon) check consistency in page 512
- (swap-anon) check consistency in page 1024
- (swap-anon) check consistency in page 1536
- (swap-anon) check consistency in page 2048
- (swap-anon) check consistency in page 2560
- (swap-anon) check consistency in page 3072
- (swap-anon) check consistency in page 3584
- (swap-anon) check consistency in page 4096
- (swap-anon) check consistency in page 4608
- (swap-anon) end
```

---
### 1.2.  문제 지점 발견 
```c
static bool anon_swap_in(struct page *page, void *kva) {

  ...
  lock_acquire(&swap_lock);
  for (int i = 0; i < SECTORS_PER_PAGE; i++){
    sector_no = (slot_idx * SECTORS_PER_PAGE) + i;
    disk_read(swap_disk, sector_no, kva + (i * DISK_SECTOR_SIZE));
  }
  lock_release(&swap_lock);
  .....
  // ❌ 문제의 지점 
  return page->uninit.page_initializer(page, VM_ANON, kva);
```
- 디버그 결과 익명 페이지가 `swap_in`하는 과정 속 `return` 문에서 종료가 되는 것을 볼 수 있다.
- `page->uninit.page_initializer(page, VM_ANON, kva);`는 프로젝트 처음부터 있던 로직이었다. 근데 이 로직이 자연스러워보여서 눈치채지 못하고 있었다.
- 일단 **`swap_in`하는 페이지는 `uninit` 페이지가 아니기** 때문에 이 **로직은 삭제**하는 것이 맞아 보인다. 

---
### 1.3.  수정 및 결과 
> 완전히 해결되지는 않았다.
> 그래도 기대 출력값이 2줄 더 생기긴 했다 (기존 1024 -> 개선 2048)

```c
static bool anon_swap_in(struct page *page, void *kva) {
  ....
  // return page->uninit.page_initializer(page, VM_ANON, kva);
  return true;
}
```

```c
Differences in `diff -u' format:
  (swap-anon) begin
  (swap-anon) write sparsely over page 0
  (swap-anon) write sparsely over page 512
  (swap-anon) write sparsely over page 1024
  (swap-anon) write sparsely over page 1536
  (swap-anon) write sparsely over page 2048
- (swap-anon) write sparsely over page 2560
- (swap-anon) write sparsely over page 3072
- (swap-anon) write sparsely over page 3584
- (swap-anon) write sparsely over page 4096
- (swap-anon) write sparsely over page 4608
- (swap-anon) check consistency in page 0
- (swap-anon) check consistency in page 512
- (swap-anon) check consistency in page 1024
- (swap-anon) check consistency in page 1536
- (swap-anon) check consistency in page 2048
- (swap-anon) check consistency in page 2560
- (swap-anon) check consistency in page 3072
- (swap-anon) check consistency in page 3584
- (swap-anon) check consistency in page 4096
- (swap-anon) check consistency in page 4608
- (swap-anon) end
```

---
### 1.4.  2차 문제 

> 아직도 출력문이 다르기 때문에 더 찾아야 된다.

*💢문제 - 디버그하는데 너무 반복이 많다* 
- 뭔가 이걸 다 반복할 수 없다.... 특이점을 찾아야 한다.

이전 버그 해결할 때 break point찍어 놓은 곳들이 `victim`과 `swap_out`로직이었는데 이걸 제거하고 갈만한 곳이 뭘까 생각해봤다.
일단 수많은 페이지를 swap_out하면 결국 swap_in도 해야할테니 swap_in도해야되니 모든 break_point를 제거하고 `swap_in`에 집중하기로 했다.
그 결과 역시나 2048페이지까지 빠르게 `swap_in`이 찍히는 것을 볼 수 있었다. 근데 또 특이한점이 발견됐다. `anon`페이지를 실컷 swap_in하다가 계속 `uniit`페이지를 `swap_in`하는 것을 볼 수 있었다.
![Pasted image 20251008212335.png](/img/user/supporter/image/Pasted%20image%2020251008212335.png)![Pasted image 20251008212424.png](/img/user/supporter/image/Pasted%20image%2020251008212424.png)

---
### 1.5.  문제 발견 

> Frame 중복 할당으로 인한 누수 

- 이렇게 몇천번 반복을 도는 테스트에서 하나하나 디버그를 하는 것은 너무 어렵다.
- 이런 테스트일수록 좀 더 근본적으로 코드를 회고해봐야한다.
	- breaking_point로 안 점은 `claim`, `swap_in`, `swap_out`을 계속해서 반복한다는 점.
	- 그리고 이 테스트 코드의 이름 자체가 swap-~ 라는 점을 볼 때 swap_in, swap_out을 위주로 회고하면 될 것이다.

![Pasted image 20251008231952.png](/img/user/supporter/image/Pasted%20image%2020251008231952.png)
- swap_in을 호출하는 쪽에서 이미 frame을 할당하고 그 내부의 kva를 swap_in인자로 넘긴 것이다
- 근데 swap_in 내에서 frame을 또 할당하고 있었다.
- 따라서, 사진의 변경사항처럼 중복 frame을 할당 

---
## 2.  swap-file

```c
/* Checks if file-mapped pages are properly swapped out and swapped in For this test, Pintos memory size is 128MB
 * First, fills the memory with with anonymous pages
 * and then tries to map a file into the page */

void test_main(void) {
  size_t handle;
  char *actual = (char *)0x10000000;
  void *map;
  size_t i;

  /* Map a page to a file */
  CHECK((handle = open("large.txt")) > 1, "open \"large.txt\"");
  // ✅ mmap
  CHECK((map = mmap(actual, sizeof(large), 0, handle, 0)) != MAP_FAILED,
        "mmap \"large.txt\"");

  /* Check that data is correct. */
  if (memcmp(actual, large, strlen(large)))
    fail("read of mmap'd file reported bad data");

  /* Verify that data is followed by zeros. */
  size_t len = strlen(large);
  size_t page_end;
  for (page_end = 0; page_end < len; page_end += 4096);
  
  for (i = len + 1; i < page_end; i++) {
    if (actual[i] != 0) {
      fail("byte %zu of mmap'd region has value %02hhx (should be 0)", i,
           actual[i]);
    }
  }

  /* Unmap and close opend file */
  munmap(map);
  close(handle);
}
```
*주석 曰*
- `file-mapped`된 `page`가 적절하게 `swap in/out`되는지 테스트하기 위함이다.
- `First, fills the memory with with anonymous pages and then tries to map a file into the page`
	- 처음에는 익명페이지로 물리 메모리를 가득 채운다.
	- 그 후 `mmap()`으로 파일을 메모리에 매핑한다

*테스트 의도 파악*
- 주석 그대로 물리 메모리가 꽉 찬 상태에서 `file-mapped`된 `page`가 적절하게 `swap in/out`되는지 원본 데이터 그대로 복구하는지 테스트

`mmap`할때 Lazy Loading한다.
근데, `memcmp`로 비교할 때 읽기 접근을 해서 Page Fault가 발생한다.
그렇게 대량의 길이로 하나하나 접근하는데 물리 메모리가 부족할 것이다 이 때 swapping과정이 잘 일어나는지 테스트 



### 2.1.  원인

> 무한 재귀 발생 -> 커널 스택 터짐 발생

왜 무한 재귀가 발생할까❓내 기존 코드를 보자
```c
static bool file_backed_swap_in(struct page* page, void* kva) {
	... 

	// ❌ 잘못된 로직 
  if (!vm_alloc_page_with_initializer(VM_FILE, page->va, true, lazy_load_segment, aux)) {
    return false;
  }

	// ❌ 잘못된 로직 
  // page claim
  if (!vm_claim_page(page->va)) {
    return false;
  }
  ....
	memset(kva + read_bytes, 0, page_zero_bytes);
  ... 
```
- 여기서 무한 루프가 왜 발생할지 생각해봤다. 잘못 구현되어 있었다.
- *`vm_do_claim_page` -> `swap_in` -> `vm_do_claim_page` 무한 재귀*
	- **애초에 `swap_in` 자체가 `vm_claim_page()` 로직을 통해 호출이 되는데 여기서 다시 `vm_claim_page()`를 호출**하고 있었다.
	- 즉, 내 초기 file_backed_swap_in은 아주~~ 잘못된 코드인 것이다.
	- 이렇게 되면 유저가 아니라 커널이 `vm_claim_page`를 호출하고 page_fault를 일으키는 문제가 발생 

### 2.2.  문제 해결 
Page Fault 시 발생하는 로직들은 여기서 호출하면 안된다.

단순히 생각하기로 했다. 
File-Backed Page는 단순히 file에서 읽어서 Load하면 된다. 그리고 페이지에서 남는 부분을 0으로 초기화하면 됨 

```c
static bool file_backed_swap_in(struct page* page, void* kva) {

  enum vm_type type = VM_TYPE(page->operations->type);
  ASSERT(type == VM_FILE);
  ASSERT(kva != NULL);
  struct file_page* file_page = &page->file;
  ASSERT(file_page->file != NULL);
  size_t page_read_bytes = file_page->page_read_bytes;
  size_t page_zero_bytes = file_page->page_zero_bytes;
  
  
  off_t read_bytes = file_read_at(file_page->file, kva, page_read_bytes, file_page->ofs);
  ASSERT(read_bytes == page_read_bytes);
  memset(kva + read_bytes, 0, page_zero_bytes);
  return true;
}
```
