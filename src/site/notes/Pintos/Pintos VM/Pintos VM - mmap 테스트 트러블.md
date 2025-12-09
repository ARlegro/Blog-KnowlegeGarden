---
{"dg-publish":true,"permalink":"/Pintos/Pintos VM/Pintos VM - mmap 테스트 트러블/","noteIcon":"","created":"2025-12-03T14:52:52.742+09:00","updated":"2025-12-09T17:19:49.271+09:00"}
---

### 0.1.  목차

- [[#테스트|테스트]]
	- [[#테스트#1.  mmap-read.c|1.  mmap-read.c]]
		- [[#1.  mmap-read.c#1.1.  기존 코드 문제 분석|1.1.  기존 코드 문제 분석]]
		- [[#1.  mmap-read.c#1.2.  개선 코드|1.2.  개선 코드]]
	- [[#테스트#2.  mmap-write.c|2.  mmap-write.c]]
		- [[#2.  mmap-write.c#2.1.  테스트 코드 및|2.1.  테스트 코드 및]]
		- [[#2.  mmap-write.c#2.2.  헷갈렸던 점|2.2.  헷갈렸던 점]]
			- [[#2.2.  헷갈렸던 점#2.2.1.  테스트의 정확한 의도 파악|2.2.1.  테스트의 정확한 의도 파악]]
			- [[#2.2.  헷갈렸던 점#2.2.2.  더티 체킹을 어디서하지?|2.2.2.  더티 체킹을 어디서하지?]]
		- [[#2.  mmap-write.c#2.3.  구현|2.3.  구현]]
			- [[#2.3.  구현#2.3.1.  첫 시도|2.3.1.  첫 시도]]
			- [[#2.3.  구현#2.3.2.  최종 시도|2.3.2.  최종 시도]]
	- [[#테스트#3.  mmap-clean.c|3.  mmap-clean.c]]
		- [[#3.  mmap-clean.c#3.1.  테스트 코드|3.1.  테스트 코드]]
		- [[#3.  mmap-clean.c#3.2.  문제의 장소|3.2.  문제의 장소]]
		- [[#3.  mmap-clean.c#3.3.  디버깅 후 사건 현장 발견|3.3.  디버깅 후 사건 현장 발견]]
		- [[#3.  mmap-clean.c#3.4.  치명적 오류 발견|3.4.  치명적 오류 발견]]
		- [[#3.  mmap-clean.c#3.5.  해결|3.5.  해결]]
	- [[#테스트#4.  mmap-inherit 😞|4.  mmap-inherit 😞]]
		- [[#4.  mmap-inherit 😞#4.1.  문제 지점 발견 - copy|4.1.  문제 지점 발견 - copy]]
		- [[#4.  mmap-inherit 😞#4.2.  문제 코드 분석|4.2.  문제 코드 분석]]
		- [[#4.  mmap-inherit 😞#4.3.  1차 해결 - file 초기화를 따로 해주기|4.3.  1차 해결 - file 초기화를 따로 해주기]]
		- [[#4.  mmap-inherit 😞#4.4.  문제 💢|4.4.  문제 💢]]
		- [[#4.  mmap-inherit 😞#4.5.  2차 해결 - 아이디어 및 구현|4.5.  2차 해결 - 아이디어 및 구현]]

---
# 테스트 
---
## 1.  mmap-read.c 
> 사실 오류가 뜨는게 정상이다.
> - 첫 mmap 테스트이고 
> - 애초에 내가 pintos를 해결해가는 패턴에서는 **완벽히 만들어 놓고 테스트를 돌리지 않기 때문**이다.
> - 나는 **뼈대만 대충 만들고 테스트 돌리면서 GitBook 보고 디버깅하는 스타일이라 오류가 뜨는 것은 당연**
> - 그래도 첫 테스트 실패인만큼 어디서 문제 생기는지 기록 해보고 싶었다.


문제는 아래의 로직에서 나왔다. 아래의 코드는 **"do_mmap ➡ Page Fault ➡ PF 핸들 ➡ Lazy_Load"** 하는 상황이다
![Pasted image 20251006123840.png](/img/user/supporter/image/Pasted%20image%2020251006123840.png)
- 분명 `aux`로 넘겨준 인자에서는 `page_read_bytes`가 4096인데 실제 읽은 bytes수는 794이다.
- 우선 생각해본 점은 2가지였다.
	1. file_reopen을 안했나❓➡ 역시나 안했었다.
	2. bytes 관련 로직이 잘못됐나❓
		 - 기존에 구현한 `do_mmap` 의 byte 로직은 `load_segment`코드에 있는 것을 살짝 수정한건데 그걸 잘못건드렸나? 생각이 들었다.

---
### 1.1.  기존 코드 문제 분석
```c
void* do_mmap(void* addr, size_t length, int writable, struct file* file,
              off_t offset) {

  size_t read_bytes = length;
  void *upage = addr;

  while (read_bytes > 0) {
    size_t page_read_bytes = read_bytes < PGSIZE ? read_bytes : PGSIZE;
    upage = pg_round_down(addr);

    /* TODO: Set up aux to pass information to the lazy_load_segment. */
    struct lazy_load_aux* aux = malloc(sizeof(struct lazy_load_aux));
    aux->file = file;
    aux->page_read_bytes = page_read_bytes;
    aux->ofs = offset;
    aux->is_writable = writable;
    if (!vm_alloc_page_with_initializer(VM_ANON, upage, writable,
                                        lazy_load_segment, aux)) {
      free(aux);
      return false;
    }

    /* Advance. */
    read_bytes -= page_read_bytes;
    offset += page_read_bytes;
    upage += PGSIZE;
  }
  return addr;
}
```

자세하게 이 부분을 디버깅하고 건네지는 값들을 보다보면 앞서 말한 문제 말고도 큰 문제들이 보인다. 보자

---
### 1.2.  개선 코드

>[!tip] 주요 개선 포인트
>1. *file_reopen*
>2. *잘못된 인자 전달 개선*
>	- ❌`upage = pg_round_down(addr);`
>	- ✅`upage = pg_round_down(upage);`
>3. *`read_bytes` 올바른 계산*
>	- `read_bytes`는 file에서 읽어야 할 byte 수이다.
>	- 하지만 만약 실제 file byte길이가 read_bytes보다 작다면 잘못처리 될 것 
>	- 따라서 
>4. *`total_zero_bytes` 추가*
>	- 애초에 `do_mmap` 로직들은 애초에 파일 부팅 시 실행되는 `load_segment`로직을 가져온 것이다.(살짝만 바꾸는 정도)
>	- 근데 처음에 zero_bytes가 필요하겠어? 라는 생각으로 지웠었는데 GitBook 가이드의 stick out 처리에 대한 글을 자세히 보니 문제가 발생할 수 있다는 생각이 들어 수정했다([Gitbook 링크](https://yjohdev.notion.site/PROJECT-3-VIRTUAL-MEMORY-d16fc8d04f4d4829b7e25691a235901c#dd46ff9de30e48c985c7eacf25384c4a))
>5. 임시 처리 : *`reopened`일 경우를 체크해서 `lazy_load` 실패 시 `file_close`처리*
>	- 일단 **임시 처리**이다.
>	- 아직 `file-backed page`에 대한 종료 처리와 swapping처리 등 완성하지 않아서 destroy라던가 exit에서 처리해야하는지 아직 모른다.
>	- 그래서 일단 `reopened`일 경우 `file_close`를 여기서 처리하는 로직을 추가했다.
>6. *번외 : 검증 로직 추가* 
>	- GitBook 다시 읽다가 예외처리하라는거 처리했다.

```c
// stick out -> 페이지 단위 작업시 페이지 끝 튀어나오는 부분 0으로 처리 필
void* do_mmap(void* addr, size_t length, int writable, struct file* file,
              off_t offset) {
	// ☑ 번외 
  ASSERT(addr != 0 || addr != NULL || length > 0 || pg_ofs(addr) == 0);

  off_t f_length = file_length(file);

	// ✅ read_bytes 개선 
  size_t read_bytes = (f_length > length) ? length : f_length;
  size_t total_zero_bytes = (f_length - offset > 0) ? f_length - read_bytes : 0;

  void* upage = addr;

  // ✅ file_reopen
  struct file* reopened_file = file_reopen(file);
  if (reopened_file == NULL) return NULL;

  // zero_bytes == 0 인 빈파일도 처리 (그냥 load_segment 로직이랑 똑같이)
  while (read_bytes > 0 || total_zero_bytes == 0) {
    size_t page_read_bytes = read_bytes < PGSIZE ? read_bytes : PGSIZE;
    size_t page_zero_bytes = PGSIZE - page_read_bytes;
    // 인자 전달 개선 : 기존 = upage = pg_round_down(addr);❌ 
    upage = pg_round_down(upage);

    /* TODO: Set up aux to pass information to the lazy_load_segment. */
    struct lazy_load_aux* aux = malloc(sizeof(struct lazy_load_aux));
    if (aux == NULL) return NULL;
    aux->file = reopened_file;
    aux->page_read_bytes = page_read_bytes;
    aux->ofs = offset;
    aux->is_writable = writable;
    // ✅ 추가한 필드들 
    aux->page_zero_bytes = page_zero_bytes;
    aux->is_reopened = true; 
    if (!vm_alloc_page_with_initializer(VM_FILE, upage, writable,
                                        lazy_load_segment, aux)) {
      free(aux);
      do_munmap(addr);
      return false;
    }

    read_bytes -= page_read_bytes;
    // ✅이것도 추가 
    total_zero_bytes -= page_zero_bytes;
    offset += page_read_bytes;
    upage += PGSIZE;
  }
  return addr;
}
```

---
## 2.  mmap-write.c 
> 테스트 파일 목적 - Dirty 여부에 따른 `file-write-back`이 일어나는지 

---
### 2.1.  테스트 코드 및 
```c
#define ACTUAL ((void *)0x10000000)

void test_main(void) {
  int handle;
  void *map;
  char buf[1024];

  /* Write file via mmap. */
  CHECK(create("sample.txt", strlen(sample)), "create \"sample.txt\"");
  CHECK((handle = open("sample.txt")) > 1, "open \"sample.txt\"");
  CHECK((map = mmap(ACTUAL, 4096, 1, handle, 0)) != MAP_FAILED,
        "mmap \"sample.txt\"");
  memcpy(ACTUAL, sample, strlen(sample));
  munmap(map);

  /* Read back via read(). */
  read(handle, buf, strlen(sample));
  CHECK(!memcmp(buf, sample, strlen(sample)),
        "compare read data against written data");
  close(handle);
}

// 기대 출력 값
check_expected (IGNORE_EXIT_CODES => 1, [<<'EOF']);
(mmap-write) begin
(mmap-write) create "sample.txt"
(mmap-write) open "sample.txt"
(mmap-write) mmap "sample.txt"
(mmap-write) compare read data against written data
(mmap-write) end
EOF
pass;
```
테스트의 흐름은 다음과 같다.
1. file을 생성
2. 생성한 파일을 열고 FD번호를 받는다.
3. 특정 주소 값에 해당 file을 mmap 시킨다.
4. 해당 주소와 매핑된 메모리에 특정 값(sample)을 memcpy로 변경한다.
5. unmap한다 
6. 다시 파일을 읽는다.
7. 읽었을 때 과연 실제 파일이 변경됐는지를 확인한다.
	- 특정 값(sample)이 생성한 파일에 반영됐는지 


---
### 2.2.  헷갈렸던 점 

---
#### 2.2.1.  테스트의 정확한 의도 파악 
> Dirty Page WriteBack 누락분을 눈치채기 힘들었다
- 디버깅을 해도 아무리 봐도 이상한 부분이 없고 정상 흐름대로 가는데 비교가 틀린다고 계속 나온다.
- 1시간 째 디버깅만 수십번 돌려보다가 답이 안나와서 **이 테스트의 목적에 대해서 GPT한테 물어봤다**. 그 결과 **file-backed page와 연결된 메모리의 변경사항이 실제 파일에 변경되는지에 대한 테스트**라고 하더라
- 그 대답을 보고 **나중에 구현해야겠다고 미뤄왔던 Dirty pabe WriteBack로직을 구현하기로 했다.**

아래는 미뤄놓은 채로 놨두었던 로직이다.
```c
// file.c 
/* Destory the file backed page. PAGE will be freed by the caller. */
// 미구현 
static void file_backed_destroy(struct page* page) {
  // dirty여부 파악하고
  struct file_page* file_page UNUSED = &page->file;
  free(page->frame);
  if (pml4_is_dirty(thread_current()->pml4, page->va)) {
  }
  pml4_clear_page(thread_current()->pml4, page->va);
}
```

---
#### 2.2.2.  더티 체킹을 어디서하지?

Pintos에서는 이 페이지가 더러워진지 체크하는 cpu관련 스켈레톤 코드들이 있다.
```c
/* Returns true if the PTE for virtual page VPAGE in PML4 is dirty,
 * that is, if the page has been modified since the PTE was
 * installed.
 * Returns false if PML4 contains no PTE for VPAGE. */
bool pml4_is_dirty(uint64_t *pml4, const void *vpage) {
  uint64_t *pte = pml4e_walk(pml4, (uint64_t)vpage, false);
  return pte != NULL && (*pte & PTE_D) != 0;
}

/* Set the dirty bit to DIRTY in the PTE for virtual page VPAGE in PML4. */
void pml4_set_dirty(uint64_t *pml4, const void *vpage, bool dirty) {
  uint64_t *pte = pml4e_walk(pml4, (uint64_t)vpage, false);
  if (pte) {
    if (dirty)
      *pte |= PTE_D;
    else
      *pte &= ~(uint32_t)PTE_D;
    if (rcr3() == vtop(pml4)) invlpg((uint64_t)vpage);
  }
}
```
- 확인하는거는 그렇다 쳐도 
- `memcpy` 같은 함수를 통해 페이지를 더럽히면 시스템 콜이 발생하지 않기 때문에 '어디서 OS가 설정해야되지?' 라는 생각이 들었다.

혹시나 어디서 해주나? 라는 생각에 전체 파일에서 `pml4_set_dirty`를 검색해도 이 함수를 호출하는 쪽은 없었다.
![Pasted image 20251007143132.png](/img/user/supporter/image/Pasted%20image%2020251007143132.png)
결국 내가 해야 한다는 뜻인가... 근데 시스템 콜이 발생하지 않는데 어디서 해야한다는 것인가 ㅠㅠ 

일단 GitBook에서 내가 놓친게 있나 확인해봤다. 그래도 아무 정보가 없다...
확인해보니 Dirty Bits에 대한 정보가 아래와 같이 적혀있었다
- x86-64 하드웨어는 페이지 교체 알고리즘을 구현에 필요한 보조 수단(= 각 페이지에 대한 페이지 테이블 엔트리에 있는 한 쌍의 비트)을 제공합니다. 
- 페이지를 읽거나 쓸 때마다, CPU는 페이지 테이블의 accessed bit를 1로 설정하고, 페이지에 쓸 때마다 dirty bit를 1로 설정합니다. 
- CPU는 이 비트들을 절대 0으로 다시 리셋하지 않지만 운영체제는 이를 0으로 리셋할 수도 있습니다.

이 부분은 VM 시작단계에서 정독할 때는 몰랐는데 지금 읽어보니 **CPU 수준에서 Dirty, Accessed는 자동으로 설정해준다는 뜻** 같다.
But 0으로 초기화하는 것과 체크하는 것은 OS가(내가) 해야하는 것 같다.
☑즉, CPU 수준에서 자동으로 dirty 처리를 해준다 ㄷㄷ 


---
### 2.3.  구현 

Dirty여부를 체크한 뒤 어떻게 실제 파일에 반영할지만 생각하면 될 것이다.
- file_write를 써야할 것이고 
- 어디서부터? 얼만큼 쓸지? 정도??

---
#### 2.3.1.  첫 시도
```c
static void file_backed_destroy(struct page* page) {
  // dirty여부 파악하고
  struct file_page* file_page UNUSED = &page->file;
  if (pml4_is_dirty(thread_current()->pml4, page->va)) {
    off_t ofs = page->file.ofs;
    size_t read_bytes = page->file.page_read_bytes;
    lock_acquire(&file_lock);
    file_write_at(page->file.file, page->frame->kva, read_bytes, ofs);
    lock_release(&file_lock);
  }
  free(page->frame);
  pml4_clear_page(thread_current()->pml4, page->va);
}
```
- page의 file_page 구조체에 있는 속성들과 실제 메모리에 있는 내용들을 가진 주소(kva)를 이용해 `file_write_at`했다.
- 또한 file관련된 로직이기에 lock을 만들고 걸었다.

결과 
```c
FAIL tests/vm/mmap-write
Kernel panic in run: PANIC at ../../userprog/exception.c:96 in kill(): Kernel bug - unexpected interrupt in kernel
Call stack: 0x8004219821 0x800421f0b0 0x800421f2aa 0x8004209912 0x8004209d6a 0x800421ad47 0x800420a9b5 0x800420af49 0x8004224dc4 0x80042242e2 0x8004223e1e 0x80042252a1 0x80042253cb 0x800421f7db 0x800421f74b 0x800421f321 0x40019e 0x400d84 0x400dd4
Translation of call stack:
0x0000008004219821: debug_panic (lib/kernel/debug.c:32)
0x000000800421f0b0: kill (userprog/exception.c:102)
0x000000800421f2aa: page_fault (userprog/exception.c:163)
0x0000008004209912: intr_handler (threads/interrupt.c:331)
0x0000008004209d6a: intr_entry (intr-stubs.o:?)
0x000000800421ad47: list_insert_ordered (lib/kernel/list.c:395 (discriminator 1))
0x000000800420a9b5: sema_down (threads/synch.c:76)
0x000000800420af49: lock_acquire (threads/synch.c:232)
0x0000008004224dc4: file_backed_destroy (vm/file.c:67)
0x00000080042242e2: vm_dealloc_page (vm/vm.c:262)
0x0000008004223e1e: spt_remove_page (vm/vm.c:141)
0x00000080042252a1: remove_related_regions (vm/file.c:151)
0x00000080042253cb: do_munmap (vm/file.c:182)
0x000000800421f7db: syscall_munmap (userprog/syscall.c:188)
0x000000800421f74b: syscall_handler (userprog/syscall.c:166)
0x000000800421f321: no_sti (syscall-entry.o:?)
0x000000000040019e: (unknown)
0x0000000000400d84: (unknown)
0x0000000000400dd4: (unknown)
FAIL
test 1/1 finish
```
- 흠... 락을 얻고나서 갑자기 sema_down??? 그러고 page_fault??
- 일단 lock을 풀어볼까??

*임시 해결* : 락을 풀자 해결이 됐다.
```c
pass tests/vm/mmap-write
PASS
test 1/1 finish

=== Test Summary ===
Passed: 1
```

---
#### 2.3.2.  최종 시도 

>[!QUESTION]  락을 어케하지??
>- `file`에 쓰기 작업이라 나중에 락이 필요한 것 같은데 
>- 답은 쉬웠다. 이번 `lock`은 `file.c`의 로직들을 위한 `lock`이라 새로 만든 `lock`이다. 하지만 `lock_init`을 하지 않고 `lock_acquire`, `lock_release`를 사용해서 문제가 발생 
>- 따라서, 아래처럼 `file_page`가 실행될 때 `lock_init`을 하도록 수정했다.
```c
struct lock file_lock;

void vm_file_init(void) {
  lock_init(&file_lock); // ✅ init 추가 
}
```
- 위처럼 `lock_init` 함수를 추가하고 기존에 쓰던 lock을 쓰고 이 테스트에 합격했다.

---
## 3.  mmap-clean.c

---
### 3.1.  테스트 코드
```c
void test_main(void) {
  static const char overwrite[] = "Now is the time for all good...";
  static char buffer[sizeof sample - 1];
  char *actual = (char *)0x54321000;
  int handle;
  void *map;

  /* Open file, map, verify data. */
  CHECK((handle = open("sample.txt")) > 1, "open \"sample.txt\"");

  CHECK((map = mmap(actual, 4096, 0, handle, 0)) != MAP_FAILED,
        "mmap \"sample.txt\"");
  if (memcmp(actual, sample, strlen(sample)))
    fail("read of mmap'd file reported bad data");

  /* Modify file. */
  CHECK(write(handle, overwrite, strlen(overwrite)) == (int)strlen(overwrite),
        "write \"sample.txt\"");

  /* Close mapping.  Data should not be written back, because we
     didn't modify it via the mapping. */
  msg("munmap \"sample.txt\"");
  munmap(map);

  /* Read file back. */
  msg("seek \"sample.txt\"");
  seek(handle, 0);
  CHECK(read(handle, buffer, sizeof buffer) == sizeof buffer,
        "read \"sample.txt\"");

  /* Verify that file overwrite worked. */
  if (memcmp(buffer, overwrite, strlen(overwrite)) ||
      memcmp(buffer + strlen(overwrite), sample + strlen(overwrite),
             strlen(sample) - strlen(overwrite))) {
    if (!memcmp(buffer, sample, strlen(sample)))
      fail("munmap wrote back clean page");
    else
      fail("read surprising data from file");
  } else
    msg("file change was retained after munmap");
}
```
대충 봤을 때는 뭐...지? 싶을 것이다. 하지만 이전에 unmap, map 관련 테스트들을 계속 뚫고 여기까지 왔다면 크게 고칠 것이 없을 것이다.
하지만 한 가지 문제에 봉착했다. 중간에 주석으로 처리된 `/* Modify file */` 부분에서 계속 에러가 났다.


---
### 3.2.  문제의 장소 
주석으로 처리된 `/* Modify file */` 부분에서 계속 에러가 났다.
```c
/* Modify file. */
  CHECK(write(handle, overwrite, strlen(overwrite)) == (int)strlen(overwrite),
        "write \"sample.txt\"");
```

```c
Acceptable output:
  (mmap-clean) begin
  (mmap-clean) open "sample.txt"
  (mmap-clean) mmap "sample.txt"
  (mmap-clean) write "sample.txt"
  (mmap-clean) munmap "sample.txt"
  (mmap-clean) seek "sample.txt"
  (mmap-clean) read "sample.txt"
  (mmap-clean) file change was retained after munmap
  (mmap-clean) end
Differences in `diff -u' format:
  (mmap-clean) begin
  (mmap-clean) open "sample.txt"
  (mmap-clean) mmap "sample.txt"
  (mmap-clean) write "sample.txt"
- (mmap-clean) munmap "sample.txt"
- (mmap-clean) seek "sample.txt"
- (mmap-clean) read "sample.txt"
- (mmap-clean) file change was retained after munmap
- (mmap-clean) end

(Process exit codes are excluded for matching purposes.)
FAIL
test 1/1 finish
```
- 출력문을 보면 write하는 곳에서 문제가 생겨 `munmap`으로 넘어가지 못하고 있는 것을 볼 수 있다.
- 오늘도 디버깅을 시작~해보자 

---
### 3.3.  디버깅 후 사건 현장 발견
> 사건 현장
> - `write` 시스템 콜 호출 ➡ file에 쓰려는 내용이 담긴 주소 검증 

주소를 검증하는 로직에는 해당 page가 `writable`불가 시 `exit`처리를 하도록 했었다.
이전에 
```c
// write 시스템 콜 내 검증 로직 
void vali_pointer(const void* user_addr, size_t size) {
  
  const uint8_t* check_ptr = user_addr;  // 현재 검사할 위치 (byte 단위 포인터)

  // 검사할 전체 영역을 페이지 단위로 나누어 한 페이지씩 접근 가능 여부를 확인
  while (byte_left > 0) {
    // 현재 check_ptr 포인터가 가리키는 주소가 사용자 영역에 있고
    // 실제로 물리 메모리에 매핑되어 있는지 확인
    if (!check_page(check_ptr)) {
      syscall_exit(-1);  // 잘못된 주소일 경우, 즉시 프로세스 종료
    }

// vali_pointer 함수에 의해 불러지는 함수    
bool check_page(const void* user_addr) {
  // || pml4_get_page(thread_current()->pml4, user_addr) == NULL
  if (user_addr == NULL || !is_user_vaddr(user_addr)) {
    return false;
  }

#ifdef VM
  // VM: spt에서 페이지를 찾아 writable 여부를 확인하기(코드/세그먼트 영역에
  // 쓰기 불가능하도록)
  struct page* p =
      spt_find_page(&thread_current()->spt, pg_round_down(user_addr));
  if (p != NULL) return p->writable; // ❌ 문제의 지점 
#endif
  return true;

}    
```
- 문제의 지점은 `file`에 `write`할 내용이 담긴 주소에 해당하는 `page`가 `writable` 금지 처리되어 있었고 이로 인해 `syscall_exit(-1)`처리 되어 있다.
- 그렇다면 2가지를 주의 깊게 봐야 된다. 
	1. 그 주소는 뭐인가?
	2. 그 페이지는 왜 writable false인가?

*☑주소의 정체* 
우선 그 주소는 테스트 코드 맨 앞에 있는 `overwrite` 배열이다.
```c
void test_main(void) {
  static const char overwrite[] = "Now is the time for all good...";
```
- 이 주소는 스택 영역에 매핑되는 변수이다.
- 근데 현재 코드 상 스택 영역은 애초에 writable = true로 설정을 해놓고 시작한다.
- 그렇다면 왜 갑자기 writable이 false?? 3.4 치명적 오류 발견으로 가보자 

---
### 3.4.  치명적 오류 발견 
`write` 시스템 콜을 자세히 보다가 뭔가 이상한 점을 발견했다.
**검증하는 `user_addr`은 읽는 영역**이고 **실제 쓰는 것은 file**인데 왜 user_addr이 쓰기 불가능을 체크하지??
```c
bool check_page(const void* user_addr) {

		...
		if (p != NULL) return p->writable;  // 읽기만할건데 writable을 왜 검증
```
사실 이 로직은 이전에 아래의 테스트 코드에서 file을 읽고 코드 영역에 저장하는 오류를 처리하기 위해 적어뒀던 것이다.  근데, 아래의 테스트랑  위랑은 시스템 콜 자체가 다르고(read, write) `check_page` 검증 로직을 공유하고 있었던 것이 화근이였다.
```c
void test_main(void) {
  int handle;
  CHECK((handle = open("sample.txt")) > 1, "open \"sample.txt\"");
  read(handle, (void *)test_main, 1);
  fail("survived reading data into code segment");
}
```

---
### 3.5.  해결 
> 문제는 현재의 테스트 코드 `mmap-clean.c`뿐만 아니라 **이전 코드 영역에 byte쓰는 테스트도 해결해야한다.**

따라서 기존에 있던 writable 검증 로직을 공통 로직에 넣지 않고 `read`시스템 콜 호출 시에만 따로 넣도록 하였다.
```c
int syscall_read(int fd, void* buffer, unsigned size) {
  vali_pointer(buffer, size);  // 사용자 버퍼가 커널에서 접근 가능한지 확인
  struct page* p = spt_find_page(&thread_current()->spt, pg_round_down(buffer));
  if (p != NULL && !p->writable){
    syscall_exit(-1);
  }
```

그 결과 두 테스트 모두 통과할 수 있었다.

---
## 4.  mmap-inherit 😞

---
### 4.1.  문제 지점 발견 - copy
> 지독하게 돌려보면서 발견했다. (이것만 디버그 시작 버튼만 50번, step-over 버튼만 수백번 돌린 듯)

이 문제는 `fork`하는 과정에서 **`load`된 페이지를 복사하고 실행시킬 때 발생**했다.
익명페이지(`VM_ANON`)를 복사하고 실행시킬 때는 문제가 발생하지 않았는데 **`file-backed page`를 실행시킬 때 문제가 발생**했다.
(`file-backed page`는 완전 마지막쯤에 실행되서 디버그하는데 오래 걸렸다)
![Pasted image 20251007212744.png](/img/user/supporter/image/Pasted%20image%2020251007212744.png)
*상황 요약* 
- `fork()` ➡ 부모 SPT 복사 시 `copy_loaded_page()` 수행  ⭐⭐
- 익명 페이지는 정상 복사 및 실행됨
- `file-backed page`만 `fault`발생

*❓이 파일 타입 페이지는 뭘까❓*
- `mmap` 할 때 만든 페이지인 걸로 추정된다.
```c
void* do_mmap(void* addr, size_t length, int writable, struct file* file,
		...
    if (!vm_alloc_page_with_initializer(VM_FILE, upage, writable,
                                        lazy_load_segment, aux)) {
      goto err_cleanup;
    }
```
- Pintos는 간단히 만들기 위해 EFL헤더 관련 페이지도 `VM_ANON`으로 하기 때문에 `VM_FILE`로 페이지가 생성되는 곳은 `do_mmap()`밖에 없다.


일단 중간 정리하자면 핵심 키워드는 `mmap()`이랑 `copy()`이다. 두 키워드를 잘 조합해서 해답을 추리해나가면 될 것 같다.

---
### 4.2.  문제 코드 분석
일단 부모의 페이지가 메모리에 loaded된 경우 copy하는 로직은 아래와 같다.
자식도 즉시 로딩해야하기 때문에 aux로 Null을 줬다. 
```c
static bool copy_loaded_page(struct supplemental_page_table* dst, struct page* src) {
  enum vm_type type = VM_TYPE(src->operations->type);
  void* va = src->va;
  struct page* dst_page = NULL;
  void* kva = NULL;

  if (!vm_alloc_page_with_initializer(type, va, src->writable, NULL, NULL)) {
    return false;
  }
  
  if (!vm_claim_page(va)) return false;
```
위의 코드 설계에 따른 💢치명적 문제는 아래 코드에서 발생했다.
```c
bool file_backed_initializer(struct page* page, enum vm_type type, void* kva) {
  /* Set up the handler */
  page->operations = &file_ops;
  struct file_page* file_page = &page->file;
  struct lazy_load_aux* aux = page->uninit.aux;

  file_page->file = aux->file;
  file_page->ofs = aux->ofs;
  file_page->page_read_bytes = aux->page_read_bytes;
  file_page->page_zero_bytes = aux->page_zero_bytes;
  file_page->is_writable = aux->is_writable;
  return true;
}
```
- 이 함수는 file_backe 페이지가 생성될 때 실행되는 초기화 함수이다.
- copy과정에서 null을 넘겨줬기에 aux는 null이다. 그럼에도 불구하고 이 초기화 함수에서는 aux가 무조건 있는 것처럼 참고 하고 있는 것을 볼 수 있다.

---
### 4.3.  1차 해결 - file 초기화를 따로 해주기 
file기반 페이지를 copy할 때 초기화를 따로 해주는 것이 좋을 것 같다고 생각이 들었다.(일단 현재 상황으로서 이게 간단함 - 나중에 리팩토링 필요할 듯? 일단은 진도가 먼저니)
```c
static bool copy_loaded_page(struct supplemental_page_table* dst,
                             struct page* src) {
  enum vm_type type = VM_TYPE(src->operations->type);
  void* va = src->va;
  struct page* dst_page = NULL;
  void* kva = NULL;

  if (!vm_alloc_page_with_initializer(type, va, src->writable, NULL, NULL)) {
    return false;
  }

  if (!vm_claim_page(va)) return false;
  dst_page = spt_find_page(dst, src->va);
  if (dst_page == NULL) return false;

  // ✅ file page 초기화 
  if (type == VM_FILE) {
    // 나중에 모듈화하던가
    struct file_page* dst_fp = &dst_page->file;
    struct file_page* src_fp = &src->file;
    dst_fp->file = file_reopen(src_fp->file);
    dst_fp->ofs = src_fp->ofs;
    dst_fp->page_read_bytes = src_fp->page_read_bytes;
    dst_fp->page_zero_bytes = src_fp->page_zero_bytes;
    dst_fp->is_writable = src_fp->is_writable;
  }

  kva = dst_page->frame->kva;
  memcpy(kva, src->frame->kva, PGSIZE);
  return true;
}
```

>[!QUESTION] 그냥 이것도 lazy_load로 하면 되지 않는가?
>어차피 lazy_load 등록해도 `vm_claim_page`를 통해 바로 할당하기 때문에 문제가 없다고 생각할 수 있다. 하지만 이런 방식의 문제 몇 가지 단점이 있다.
>1. *더러워진 페이지 처리* 
>	- 부모의 페이지 중 dirty page는 아직 디스크에 있는 file에 반영되지 않은 상태이다.
>	- 근데 단순히 file을 읽어서 load하는 방식으로는 완벽히 부모의 상태를 copy할 수 없다.
>	  
>2. file I/O 성능문제
>	- 이미 부모가 file로부터 데이터를 메모리에 올라온 상태인데 그걸 또 `read`시스템 콜을 통해 불러들어온다면 성능이 별로 좋을 것 같지 않다.

>[!tip] 좋은 방법 : 즉시 로딩 + `memset`
>- 초기 메모리 세팅 없이 `FILE-BACKED`이든 `ANON` 타입이든 즉시 로딩을 한다. 그 후 **`COPY`한 부모의 페이지로부터 memory 복사**를 한다.
>- 단순 에러 해결만이 아니라 성능적으로도 이득이다('디스크 비용 > 메모리 접근 비용' 이기 때문)


---
### 4.4.  문제 💢

> 엄한 곳에서 문제가 발생했다.

현재 copy하는 단계를 생각해보면 아래와 같다.
1. 페이지 예약 - `vm_alloc_page_with_initializer`
2. 페이지 할당 - `vm_alloc`
	- 여기서 페이지 타입별 initializer 실행

근데 file-backed page타입의 `initializer`는 아래와 같았다.
```c
bool file_backed_initializer(struct page* page, enum vm_type type, void* kva) {
  /* Set up the handler */
  page->operations = &file_ops;
  struct file_page* file_page = &page->file;
  struct lazy_load_aux* aux = page->uninit.aux; // ❌문제의 지점 

  file_page->file = aux->file; // ❌ 여기서 바로 하면 null 문제 발생 가능
	file_page->ofs = llaux->ofs;
  ....
  return true;
```
- 애초에 `loaded`된 페이지의 `uninit.aux`는 없다. 
- fork 시 `file-backe-page`를 copy할 때는 `lazy_load_aux`를 넘기지 않고 `NULL`을 넘긴다. 따라서 `page->unit.aux = NULL`이라서 문제인 것이었다.
- *그렇다면 aux를 넘겨줘야 하나❓*
	- 이미 부모의 loaded된 페이지는 `lazy_load`단계에서 `aux`는 썼고, `free`처리를 한 후 `FILE-BACKED` or `ANON`타입으로 바뀐 것이다.
	- 따라서 aux를 넘겨줄려면 부모 

```c
bool lazy_load_segment(struct page* page, void* aux) {

  struct lazy_load_aux* llaux = (struct lazy_load_aux*)aux;
  // 기타 로직들 진행 
  ....
  
	free(aux); // ✅ aux는 free됨 
  return true;


struct page {
	...
  /* 타입별 데이터는 union에 묶여 있다.
   * 각 함수는 현재 union 타입을 자동으로 감지한다. */
  union {
    struct uninit_page uninit;
    struct anon_page anon;
    struct file_page file;
...
```

---
### 4.5.  2차 해결 - 아이디어 및 구현
> 로직 피하기 - `file_page` 필드값 위치 수정 (임시, 나중에 수정 예정)

*✅해결 아이디어 *
- `lazy_load_aux`를 넘기지 않는데 그렇다면 초기화를 어떻게 할지가 의문이었다.
- 일단은 임시로 `lazy_load_segment`에서 `file_page` 필드값에 대한 초기화를 담당하고 
- `lazy_load_segment`로직이 닿지 않는 `copy_loaded_page`에서는 직접 초기화했다(추가로 file을 `reopen`)
- 이에 따라 `file-backed-initializer`도 수정했다.

1. `lazy_laod_segment`에서 초기화
	```c
	  bool lazy_load_segment(struct page* page, void* aux) {
			  struct lazy_load_aux* llaux = (struct lazy_load_aux*)aux;
			  struct file_page* file_page = &page->file;
			  //struct lazy_load_aux* aux = page->uninit.aux;
			  file_page->file = llaux->file;
			  file_page->ofs = llaux->ofs;
			  file_page->page_read_bytes = llaux->page_read_bytes;
			  file_page->page_zero_bytes = llaux->page_zero_bytes;
			  file_page->is_writable = llaux->is_writable;  
	```

2. `file-backed-initializer` 수정 
```c
bool file_backed_initializer(struct page* page, enum vm_type type, void* kva) {
  /* Set up the handler */
  page->operations = &file_ops;
  // struct file_page* file_page = &page->file;
  // struct lazy_load_aux* aux = page->uninit.aux;

  // todo : 나중에 refactoring 시 여기서 초기화하도록 하기 (lazy_load_segment,
  // copy에서 하지말고) file_page->file = aux->file; file_page->ofs = aux->ofs;
  // file_page->page_read_bytes = aux->page_read_bytes;
  // file_page->page_zero_bytes = aux->page_zero_bytes;
  // file_page->is_writable = aux->is_writable;
  return true;
}
```
