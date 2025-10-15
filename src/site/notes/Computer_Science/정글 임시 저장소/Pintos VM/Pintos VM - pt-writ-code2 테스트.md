---
{"dg-publish":true,"permalink":"/Computer_Science/정글 임시 저장소/Pintos VM/Pintos VM - pt-writ-code2 테스트/","noteIcon":"","created":"2025-10-04T21:19:42.536+09:00","updated":"2025-10-15T16:24:40.883+09:00"}
---



### 0.1.  목차

- [[#1.  pt-write-code2|1.  pt-write-code2]]
	- [[#1.  pt-write-code2#1.1.  테스트 코드|1.1.  테스트 코드]]
	- [[#1.  pt-write-code2#1.2.  문제 💢|1.2.  문제 💢]]
	- [[#1.  pt-write-code2#1.3.  추측 및 해결|1.3.  추측 및 해결]]
- [[#2.  pt-grow-stk-sc|2.  pt-grow-stk-sc]]
	- [[#2.  pt-grow-stk-sc#2.1.  테스트 코드|2.1.  테스트 코드]]
	- [[#2.  pt-grow-stk-sc#2.2.  문제|2.2.  문제]]
	- [[#2.  pt-grow-stk-sc#2.3.  분석 및 해결|2.3.  분석 및 해결]]
	- [[#2.  pt-grow-stk-sc#2.4.  일부 해결 후 추가 오류|2.4.  일부 해결 후 추가 오류]]
		- [[#2.4.  일부 해결 후 추가 오류#2.4.1.  분석|2.4.1.  분석]]
		- [[#2.4.  일부 해결 후 추가 오류#2.4.2.  ✅해결|2.4.2.  ✅해결]]
	- [[#2.  pt-grow-stk-sc#2.5.  추가 문제 발생 💢|2.5.  추가 문제 발생 💢]]
	- [[#2.  pt-grow-stk-sc#2.6.  다른 해결들|2.6.  다른 해결들]]
	- [[#2.  pt-grow-stk-sc#2.7.  추가 문제|2.7.  추가 문제]]
	- [[#2.  pt-grow-stk-sc#2.8.  최종 해결|2.8.  최종 해결]]

## 1.  pt-write-code2

### 1.1.  테스트 코드 
```c
/* Try to write to the code segment using a system call.
   The process must be terminated with -1 exit code. */
void test_main(void) {
  int handle;
  
  CHECK((handle = open("sample.txt")) > 1, "open \"sample.txt\"");
  read(handle, (void *)test_main, 1);
  fail("survived reading data into code segment");
}
```
*주석 曰*
- 이 테스트는 read 시스템 콜을 이용해서 code segment에 쓰려고 시도하는 것이다.
- 이러한 시도 시 프로세스는 반드시 종료 코드 -1과 함께 종료해야 한다.

### 1.2.  문제 💢
```c
Selected tests: pt-write-code2

Running pt-write-code2 in batch mode... 
$ pintos -m 20 --fs-disk=10 -p tests/vm/pt-write-code2:pt-write-code2 -p ../../tests/vm/sample.txt:sample.txt --swap-disk=4 -- -q -f run 'pt-write-code2'

FAIL tests/vm/pt-write-code2
run: survived reading data into code segment: FAILED
FAIL
test 1/1 finish

=== Test Summary ===
Passed: 0
Failed: 1
```
현재 내 코드 상으로는 code segment 영역에 정상적으로 read가 되어 그 다음 문장인 `fail("survived reading data into code segment");`에 도착했다. 이로 인해 테스트가 fail이 됐다.

### 1.3.  추측 및 해결 
테스트 코드를 보면 `read` 시스템 콜 시 오류를 내야 하는 것 같다.
syscall_read시에 `buffer`로 넘어온 **주소 검증을 좀 더 강화해야** 할 듯?
```C
// ✅기존 valid_poitner 로직 중 check_page 로직 수정
void vali_pointer(const void *user_addr, size_t size) {
	while (byte_left > 0) {
	    if (!check_page(check_ptr)) {
	      syscall_exit(-1);  // 잘못된 주소일 경우, 즉시 프로세스 종료
	    }
	    ....	    
}

.....
bool check_page(const void *user_addr) {
	// 기존 검증 로직 
  if (user_addr == NULL || !is_user_vaddr(user_addr) || pml4_get_page(thread_current()->pml4, user_addr) == NULL){
    return false;
  }
#ifdef VM
		// ✅ 추가로직 
    // VM: spt에서 페이지를 찾아 writable 여부를 확인하기(코드/세그먼트 영역에 쓰기 불가능하도록)
    struct page *p = spt_find_page(&thread_current()->spt, pg_round_down(user_addr));
    if (p != NULL) return p->writable;
#endif
  return true;
}
```
- 기존 `valid_poitner` 로직 중 `check_page` 로직 수정
- 그 주소가 가리키는 페이지가 `writable`이 `false`일 경우 buffer를 쓰지 못 하도록 하는 검증 로직을 추가했다.


