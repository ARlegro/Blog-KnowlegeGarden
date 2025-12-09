---
{"dg-publish":true,"permalink":"/Computer_Science/Virtual_Memory/동적 메모리(2) - malloc/","noteIcon":"","created":"2025-12-03T14:52:46.010+09:00","updated":"2025-12-09T17:19:42.146+09:00"}
---


### malloc() 개념 

> malloc()을 통한 동적 메모리 할당 
```c
#include <stdlib.h> // stdlib에 내장된 함수 

int main(){
  malloc(sizeof(int)); // 파라미터 숫자 단위 = Byte
}
```
- *반환 값*
	- 성공 시 = **첫 번째 바이트를 가리키는 포인터** 
	- 실패 시 = `NULL`
- *기본 반환 타입 : `void*`*
	- 이유 : `mallo`이 메모리 할당만 하는거지 그 할당으로 **뭐가 저장되는지(유형, 내용 등)는 알지 못하기에** 
	- 만약 다른 타입으로 반환 받고 싶다면❓`int *ptr = (int *) malloc(4)`

```c
#include <stdlib.h>
#include <stdio.h>
  
int main(){
  int i, n;
  printf("몇 개의 숫자를 입력하실건가요?");
  scanf("%d", &n);
  int *p = malloc(n*sizeof(int));

  if (p == NULL){
    printf("메모리 할당 실패\n");
    exit(1);
  }

  //printf("메모리 주소: %p\n", p);
  for (i = 0; i < n; i++){
    printf("%d번째 숫자를 입력하세요: ", i+1);
    scanf("%d", &p[i]); //p가 가리키는 배열에서 i번째 원소의 주소에 숫자 입력
  }

  for (i = 0; i < n; i++){
    printf("%d번째 숫자: %d\n", i+1, p[i]);
  }
  free(p);
}
```
- 만약 malloc을 통해 p에 배정된 메모리 범위를 넘어가면 이상한 숫자가 있을 것 

>[!tip] &p[i]  == p + i
>- p는 포인터이기에 값이 아니라 주소를 포함한다.
>- 따라서 단순 연산을 통해서도 올바른 주소에 접근할 수 있다.

---
## malloc()은 어떻게 메모리를 가져오는가

`malloc()`은 내부적으로 **2개의 시스템 콜을 쓰면서 메모리를 가져온다.**
- `sbrk/brk`(연속 힙) : 작은/중간 크기 청크
- `mmap/munmap`(매핑 기반) : 큰 청크 

---
### 1. sbrk

---
#### 개념 
> Program Break(힙의 상단)를 뒤로 밀어 연속 힙을 확장(Heap 메모리가 커짐) 

![Pasted image 20250822134917.png](/img/user/supporter/image/Pasted%20image%2020250822134917.png)
- 원래, Program Break 위 부분을 읽는 것은 불가능했다. 
- 그런데 `sbrk(or brk)`는 **힙 상단을 확장**해줘서 이전에 **접근이 불가능했던 곳을 접근할 수 있게** 해줌 
- 라이브러리 내부에서는 여전히 쓰이지만 - 직접 코드로는 쓰지말기(deprecated)
	- 최근 Unix계열에서는 **직접 사용 시  `mmap()`사용 권장**.(이유는 뭐 자세히 ㄴㄴ)
	- 새로운 프로그램 시 `mmap()`방식을 권고

---
#### 번외 : 테스트
```c 
#include <stdio.h>
#include <stdlib.h>  

int main(){
  for (int i=0; i< 5 ; i++){
    char *ptr = malloc(5000);
    printf("메모리 얻었다!! 주소 = %p\n", ptr);
  }
}
```

위의 컴파일 한 뒤 프로그램이 어떤 시스템 콜을 사용하는지 추적 `strcae ./파일명`
```c
➜  test strace ./sbrk-test
execve("./sbrk-test", ["./sbrk-test"], 0x7fff32cc3660 ...

// 최근 Program Break확인 
brk(NULL)                               = 0x5633f174d000 

// malloc과 무관한 매핑 
mmap(NULL, 8192, .....)( = 0x7f7d8f30e000
....
mmap(NULL, 21759, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f7d8f308000
.....

// 여기서부터 mmap : libc.so.6(라이브러리)을 메모리에 올리며 .text, .data 등의 세그먼트별로 매핑하는 과정 (for문 개수랑 상관 ❌)
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
mmap(NULL, 2170256, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f7d8f0f6000

mmap(0x7f7d8f11e000, 1605632, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x28000) = 0x7f7d8f11e000

mmap(0x7f7d8f2a6000, 323584, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1b0000) = 0x7f7d8f2a6000

mmap(0x7f7d8f2f5000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1fe000) = 0x7f7d8f2f5000

mmap(0x7f7d8f2fb000, 52624, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f7d8f2fb000
close(3)                                = 0

// 익명 매핑 : malloc과 무관 
mmap(NULL, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f7d8f0f3000

.....
munmap(0x7f7d8f308000, 21759)           = 0

brk(NULL)                               = 0x5633f174d000 // 힙 끝 조회 
brk(0x5633f176e000)                     = 0x5633f176e000 // 힙 확장

// 여기서부터는 write 
write(1, "\353\251\224\353\252\250\353\246\254 \354\226\273\354\227\210\353\213\244!! \354\243\274\354\206\214 = 0"..., 46메모리 얻었다!! 주소 = 0x5633f174d2a0
) = 46
write(1, "\353\251\224\353\252\250\353\246\254 \354\226\273\354\227\210\353\213\244!! \354\243\274\354\206\214 = 0"..., 46메모리 얻었다!! 주소 = 0x5633f174ea40
) = 46
write(1, "\353\251\224\353\252\250\353\246\254 \354\226\273\354\227\210\353\213\244!! \354\243\274\354\206\214 = 0"..., 46메모리 얻었다!! 주소 = 0x5633f174fdd0
) = 46
write(1, "\353\251\224\353\252\250\353\246\254 \354\226\273\354\227\210\353\213\244!! \354\243\274\354\206\214 = 0"..., 46메모리 얻었다!! 주소 = 0x5633f1751160
) = 46
write(1, "\353\251\224\353\252\250\353\246\254 \354\226\273\354\227\210\353\213\244!! \354\243\274\354\206\214 = 0"..., 46메모리 얻었다!! 주소 = 0x5633f17524f0
) = 46
```
설명은 주석 참고 
- *`brk` 명령어*
	- *brk(NULL)* : 현재 힙 끝 주소(Program Break)를 확인하는 것 
	- *brk(주소 x)* : 주소 x 만큼힙 끝을 늘리기 
- 현재 위의 출력창은 코드 상의 malloc과 무관한 `mmap()`만 있다.
	- 이유 : 너무 작은 블록을 할당해서 그렇다. malloc(5000)은 너무 작음 

`malloc(200 *1024)` 로 진행한 결과 
```c
// ... 위와 거의 동일

// library 접근하며 mmap을 통해 메모리 매핑 
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
...
mmap(NULL, 2170256, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f6a39ad6000
mmap(....)
mmap(....)
mmap(....)
mmap(....)

...
munmap(0x7f6a39ce8000, 21759)           = 0

...
brk(NULL)                               = 0x560c2a492000
brk(0x560c2a4b3000)                     = 0x560c2a4b3000

//📘 malloc에 의해서 할당된 1번째 큰 청크 
mmap(NULL, 208896, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f6a39aa0000
...
write(1, "\353\251\224\353\252\250\353\246\254 \354\226\273\354\227\210\353\213\244!! \354\243\274\354\206\214 = 0"..., 46메모리 얻었다!! 주소 = 0x7f6a39aa0010
) = 46

//📘 malloc에 의해서 할당된 2번째 큰 청크 
mmap(NULL, 208896, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f6a39a6d000
write(1, "\353\251\224\353\252\250\353\246\254 \354\226\273\354\227\210\353\213\244!! \354\243\274\354\206\214 = 0"..., 46메모리 얻었다!! 주소 = 0x7f6a39a6d010
) = 46

//📘 malloc에 의해서 할당된 3번째 큰 청크 
mmap(NULL, 208896, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f6a39a3a000
write(1, "\353\251\224\353\252\250\353\246\254 \354\226\273\354\227\210\353\213\244!! \354\243\274\354\206\214 = 0"..., 46메모리 얻었다!! 주소 = 0x7f6a39a3a010
) = 46

//📘 malloc에 의해서 할당된 4번째 큰 청크 
mmap(NULL, 208896, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f6a39a07000
write(1, "\353\251\224\353\252\250\353\246\254 \354\226\273\354\227\210\353\213\244!! \354\243\274\354\206\214 = 0"..., 46메모리 얻었다!! 주소 = 0x7f6a39a07010
) = 46

//📘 malloc에 의해서 할당된 5번째 큰 청크 
mmap(NULL, 208896, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f6a399d4000
write(1, "\353\251\224\353\252\250\353\246\254 \354\226\273\354\227\210\353\213\244!! \354\243\274\354\206\214 = 0"..., 46메모리 얻었다!! 주소 = 0x7f6a399d4010
) = 46
exit_group(0) 
```
>[!tip] 정확히 for 반복 횟수(5번)만큼 `mmap()`이 추가로 생겼다
- 아래 부분을 보면 `malloc()`에 의해서 호출된 `mmap()`이 있다.
- 이유 : **큰 메모리의 할당은 힙이 아니라 개별 `mmap()`** 으로 줌

---
### 2. mmap() ⭐(나중에 합치기)

합칠 내용 : [[Computer_Science/Virtual_Memory/메모리 매핑\|메모리 매핑]]

---
#### 개념 
> **익명의 메모리 영역을 할당 받을 때**(or 파일을 메모리 매핑) 사용되는 System Call
> - 단순 메모리 할당을 넘어,
> - OS의 메모리 관리 메커니즘(가상 메모리)을 활용한다.

이번 글에서는 파일 매핑보다는 익명 매핑을 설명할 것 (파일 매핑 : [[Computer_Science/Virtual_Memory/메모리 매핑\|메모리 매핑]])

*익명 매핑(Anonymouse Mapping)*
- **순순한 메모리 영역을 할당 받을 때 사용**하는 매핑 
- 프로세스 간 **공유 메모리 구현**  시 유용
- 프로세스 간 **통신 가능**  
	- 동일한 메모리 영역을 사용함으로써 데이터 공유가 가능 

---
#### 개념 다지기{ vs sbrk()}
- 커널로부터 메모리를 요청한다는 점에서 `sbrk()`와 같지만 몇 가지 다르다
- `mmap()`은 **주소 공간에 더 많은 메모리가 사용 가능하도록**<br>![Pasted image 20250822151246.png](/img/user/supporter/image/Pasted%20image%2020250822151246.png)

| 특징            | `sbrk()` - 레거시로 간주됨                                              | `mmap()`                                                                |
| ------------- | ---------------------------------------------------------------- | ----------------------------------------------------------------------- |
| 원리            | **Program Break**의 위치를 조정하여 heap 메모리 영역을 확장                      | **가상 메모리 주소 공간**에 새로운 메모리 영역을 매핑                                        |
| **메모리 할당 방식** | 기존 힙 영역의 끝에서부터 메모리를 **연속적으로 할당**                                 | 주소 공간의 어떤 곳에든 **비연속적인 메모리 영역을 할당**할 수 있다                                |
| **유연성**       | **제한적** Cuz 힙 영역의 크기만 조절 가능(만약 크기 늘리려는데 그 공간에 다른 주소가 매핑되었으면 못 함) | **매우 유연** Cuz 파일 내용 전체를 메모리에 올리거나, 특정 크기의 익명 메모리를 할당하는 등 **다양한 옵션을 제공** |

---
#### 파라미터 분석 

내부 mmap() 구현 Param
```c
extern void *mmap (void *__addr, size_t __len, int __prot,
         int __flags, int __fd, __off_t __offset) __THROW;
```
1. `*__addr`
	- 메모리가 어디에 매핑됐으면 좋겠다는 **힌트** (반드시는 X)
	- `NULL` 주입 시 : 커널이 알아서 적절한 주소 선택 (대부분 `NULL` 사용)
	  
2. `len`, `offset`
	- 어떤 영역을 메모리로 매핑할지 결정 
		![Pasted image 20250821225300.png](/img/user/supporter/image/Pasted%20image%2020250821225300.png)
		- `length` : 블록의 크기. 매핑 할 메모리 크기(단위 : bytes)
		- `offset` 
			- 매핑할 시작 지점을 지정하는 offset. 
			- **반드시 Page Size의 배수여야** 함 ⭐ Cuz Paging 시스템을 잘 유지하기 위해 
			- 익명 매핑 시에는 보통 0 
			- 파일 매핑 시에는 파일 안에서 시작할 위치 
			  
3. `__prot`
	- 매핑된 영역의 **접근 권한 지정** 
		- 읽기 전용(`PROT_READ`) 
		- 쓰기 전용(`PROT_WIRTE`)
		- 실행 가능(`PROT_EXEC`)
		- 접근 불가(`PROT_NONE`)
		  
4. `__flags`
	- 커널에게 "**이 메모리를 어떻게 관리하고, 어떻게 공유하기를 원하는지**"설명하는 flags
	- *대표적인 성질*
		1. *공유 여부*
			- `MAP_SHARED` : 매핑된 메모리 변경사항이 프로세스간 공유
			- ⭐`MAP_PRIVATE` : 매핑된 메모리 변경 → 프로세스 개인 사본으로 (Copy-on-Write)(참고 : [[Computer_Science/Virtual_Memory/메모리 매핑\|메모리 매핑]]) 
				- `malloc()`에서 익명 메모리 할당 시 이 옵션 씀 
				  
		2. *매핑된 소스* 
			- `MAP_ANONYMOUS` : 파일과 연결하지 않고 익명 페이지들로 가져옴 
			- `malloc()`에서 큰 페이지 요청 시 씀 
			  			  
		3. *주소 제어* 
			- `MAP_Fixed` : 주소를 강제로 고정할지  ex. `__addr` 위치에 반드시 매핑.(기존 매핑 있으면 덮어쓰기 -> 위험 💢)

5. `fd` 
	- 매핑할 파일 
	- 익명 매핑이면 무조건 `-1`



>[!EXAMPLE] 번외 : C에서 페이징 사이즈 얻는 법
1. `sysconf(_SC_PAGESIZE)`
	```c
	#include <stdio.h>
	
	#include <sys/mman.h> ✅
	void main() {
		int pagesize = sysconf(_SC_PAGESIZE); ✅
		printf("pagesize = %d\n", pagesize); //pagesize = 4096
	```
2. `getpagesize()` 함수 활용 리눅스 버전 
	```c
	#include <stdio.h>
	#include <unistd.h>
	void main() {
		  int linux_pagesize = getpagesize(); ✅
		  printf("linux_pagesize = %d\n", linux_pagesize);  //linux_pagesize = 4096
	```

Next : [[Computer_Science/Virtual_Memory/동적 메모리(3) - ~적 가용 리스트\|동적 메모리(3) - ~적 가용 리스트]]
