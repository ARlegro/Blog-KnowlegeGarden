---
{"dg-publish":true,"permalink":"/Pintos/Pintos User Prog/Pintos U.P - read_normal 테스트/","noteIcon":"","created":"2025-12-03T16:03:22.663+09:00","updated":"2025-12-03T16:10:47.975+09:00"}
---


## 1.  read_normal 실패 

### 1.1.  오류 

```bash
Selected tests: read-normal Running read-normal in batch mode... $ pintos --fs-disk=10 -p tests/userprog/read-normal:read-normal -p ../../tests/userprog/sample.txt:sample.txt -- -q -f run 'read-normal' 

FAIL tests/userprog/read-normal run: size of sample.txt (8) differs from expected (373):
FAILED 
FAIL 
test 1/1 finish
```


### 1.2.  추리 과정 및 해결 
다른 read관련 test는 통과하는데 이것만 통과가 계속 안됐다.
주의해서 봐야 될 문자는  `run: size of sample.txt (8) differs from expected (373):`이거다. size가 이상하게 전달되고 있는 것 같다. 
하지만 이상할게 없는게 나의 read 로직에서는 Pintos에서 제공중인 스켈레톤 함수(`file_read()`)를 사용하고 있기에 전혀 이상할게 없었다.
```c
int read (int fd, void *buffer, unsigned size) {
  lock_acquire(&filesys_lock);
  int n = file_read(to_read_file, buffer, (off_t) size);
  lock_release(&filesys_lock);
  return n;
```

뭐지?? 하면서 디버그를 하다가 read관련 테스트에서 `check_file_handle`이라는 메서드를 통과하고 있는 것을 보았고 여기서 다른게 밝혀졌다. 초반 시작에 `filesize(fd)`라는 의문의 함수가 호출이되고 그 다음 if문에서 오류가 났었다.

```c
void
check_file_handle (int fd,
                   const char *file_name, const void *buf_, size_t size) 
{
  const char *buf = buf_;
  size_t ofs = 0;
  size_t file_size;

  /* Warn about file of wrong size.  Don't fail yet because we
     may still be able to get more information by reading the
     file. */
  file_size = filesize (fd);
  if (file_size != size)
    msg ("size of %s (%zu) differs from expected (%zu)",
          file_name, file_size, size);

  /* Read the file block-by-block, com
```
이 함수를 추적한 결과 `syscall.c`에 있는 시스템 콜 종류들 중 하나였다.<br>
![Pasted image 20250915010540.png](/img/user/supporter/image/Pasted%20image%2020250915010540.png)
따라서 `read-normal` 테스트를 통과하기 위해서는 `SYS_FILESIZE`라는 시스템 콜 핸들러를 만들어야 했고 나는 아래와 같이 만들었다.
```C
void syscall_handler (struct intr_frame *f UNUSED) {

  int sys_number = f->R.rax;
  switch (sys_number){
    case SYS_FILESIZE:{

      int fd = (int) f->R.rdi;
      f->R.rax = get_file_length(fd);
      break;
    }   
.....
}   

uint64_t get_file_length(int fd) {
	struct file *file_ptr = find_file_by_fd(fd);
	if (file_ptr == NULL){
		return -1;
	}
	
	lock_acquire(&filesys_lock);
	off_t size = file_length(find_file_by_fd(fd));
	lock_release(&filesys_lock);
	return (uint64_t)size;
}    
```
이전에 file관련 `syscall`함수들을 구현할 때 `file.c`에서 `file_length`라는 스켈레톤 함수가 있는걸 봤고 이걸 쓸라나?라는 생각이 들었는데 여기서 쓰는 거였구나~
이제 다음으로 ㄱ 