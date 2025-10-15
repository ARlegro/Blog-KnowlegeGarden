---
{"dg-publish":true,"permalink":"/Pintos/Pintos VM/Pintos VM - 구조체 및 관련 개념 정리/","noteIcon":"","created":"2025-09-27T10:19:19.108+09:00","updated":"2025-10-15T16:21:48.744+09:00"}
---

Pintos VM 구조체 및 관련 개념 정리에 관한 글 


### 0.1.  목차

- [[#1.  pml4|1.  pml4]]
	- [[#1.  pml4#1.1.  구조체 분석|1.1.  구조체 분석]]
	- [[#1.  pml4#1.2.  4-Level Paging 개념|1.2.  4-Level Paging 개념]]
	- [[#1.  pml4#1.3.  4-Level Paging 사용 이유|1.3.  4-Level Paging 사용 이유]]
	- [[#1.  pml4#1.4.  CPU가 어떻게 PML4를 사용할까?|1.4.  CPU가 어떻게 PML4를 사용할까?]]
- [[#2.  Frame|2.  Frame]]
- [[#3.  Page|3.  Page]]
- [[#4.  Page Table vs Supplemental Page Table(SPT)|4.  Page Table vs Supplemental Page Table(SPT)]]
	- [[#4.  Page Table vs Supplemental Page Table(SPT)#4.1.  Preview - 간단 차이 비교|4.1.  Preview - 간단 차이 비교]]
	- [[#4.  Page Table vs Supplemental Page Table(SPT)#4.2.  Page Table|4.2.  Page Table]]
		- [[#4.2.  Page Table#4.2.1.  페이지 테이블 쓰는 이유 in VM|4.2.1.  페이지 테이블 쓰는 이유 in VM]]
		- [[#4.2.  Page Table#4.2.2.  PTE|4.2.2.  PTE]]
	- [[#4.  Page Table vs Supplemental Page Table(SPT)#4.3.  SPT|4.3.  SPT]]
		- [[#4.3.  SPT#4.3.1.  개념|4.3.1.  개념]]
		- [[#4.3.  SPT#4.3.2.  어떻게 활용 가능한가❓⭐|4.3.2.  어떻게 활용 가능한가❓⭐]]
		- [[#4.3.  SPT#4.3.3.  실제 OS에서 SPT|4.3.3.  실제 OS에서 SPT]]
	- [[#4.  Page Table vs Supplemental Page Table(SPT)#4.4.  궁금증 : PTE에 메타데이터가 있는데 SPT가 왜 필요❓|4.4.  궁금증 : PTE에 메타데이터가 있는데 SPT가 왜 필요❓]]
	- [[#4.  Page Table vs Supplemental Page Table(SPT)#4.5.  VM 핵심 - Page Fault 처리 흐름|4.5.  VM 핵심 - Page Fault 처리 흐름]]
- [[#5.  Page 타입과 타입별 특징|5.  Page 타입과 타입별 특징]]
	- [[#5.  Page 타입과 타입별 특징#5.1.  익명 페이지|5.1.  익명 페이지]]
		- [[#5.1.  익명 페이지#5.1.1.  특징|5.1.1.  특징]]
		- [[#5.1.  익명 페이지#5.1.2.  주 사용처|5.1.2.  주 사용처]]
	- [[#5.  Page 타입과 타입별 특징#5.2.  file-backed 페이지|5.2.  file-backed 페이지]]
		- [[#5.2.  file-backed 페이지#5.2.1.  개념|5.2.1.  개념]]
		- [[#5.2.  file-backed 페이지#5.2.2.  특징|5.2.2.  특징]]
		- [[#5.2.  file-backed 페이지#5.2.3.  주 사용처|5.2.3.  주 사용처]]
- [[#6.  Page_Operations|6.  Page_Operations]]


## 1.  pml4
---
### 1.1.  구조체 분석 
```c
uint64_t *pml4;                     /* Page map level 4 */
```
- 개념 : `pml4`는 4-level Paging의 첫 번째 레벨인 Page Map Level 4 Table의 시작주소를 가리킨다.
- 즉, 4-Level-Paging 아키텍쳐에서 가상 주소 변환의 뿌리 역할
- Pintos에서는
	- `struct thread` 내부에 `pml4`필드가 있다.
	- 각 thread(실제로는 process)는 자신만의 독립된 페이지 테이블 구조를 가지며 pml4도 독립적이다.

---
### 1.2.  4-Level Paging 개념 
x86-64 linux 환경에서는 **4-Level Paging** 기법을 사용하여 **가상주소(VA)를 물리 주소(PA)로 변환**한다. 이는 4개의 Level을 가진 page table을 사용한 것. 
- *4-level이나 되는 이유*❓
	- **가상 주소 공간**이 64bit인데 이를 **공간적으로 원활히 관리하기 위해서는 4단계의 계층구조가 필요**했다.
- *4단계 계층 구조* 
	- 각 엔트리는 다음 레벨의 테이블 주소를 가리킨다
	- 마지막 레벨의 엔트리(PT의 Entry, PTE)는 실제 물리 페이지 프레임의 주소를 가리킴 

| Level | 이름                           | 약칭   | 인덱스 비트 수 | 역할                                         |
| ----- | ---------------------------- | ---- | -------- | ------------------------------------------ |
| 1     | Page Map Level 4             | PML4 | 9비트      | 최상위 디렉터리. 전체 주소 공간을 512개의 영역(512 GB씩)으로 분할 |
| 2     | Page Directory Pointer Table | PDPT | 9비트      |                                            |
| 3     | Page Directory               | PD   | 9비트      |                                            |
| 4     | Page Table                   | PT   | 9비트      |                                            |
| —     | Page Offset                  | —    | 12비트     | 실제 페이지 내부 오프셋                              |

```C
/* pte.h */

/* Functions and macros for working with x86 hardware page tables.
 * See vaddr.h for more generic functions and macros for virtual addresses.
 *
 * Virtual addresses are structured as follows:
 *  63          48 47            39 38            30 29            21 20 12 11 0
 * +-------------+----------------+----------------+----------------+-------------+------------+
 * | Sign Extend |    Page-Map    | Page-Directory | Page-directory | Page-Table
 * |  Physical  | |             | Level-4 Offset |    Pointer     |     Offset
 * |   Offset    |   Offset   |
 * +-------------+----------------+----------------+----------------+-------------+------------+
 *               |                |                |                | | |
 *               +------- 9 ------+------- 9 ------+------- 9 ------+----- 9
 * -----+---- 12 ----+ Virtual Address
 */
```

		 
![Pasted image 20251013092924.png](/img/user/supporter/image/Pasted%20image%2020251013092924.png)
- `PML4E` : PML4 Offset 시작주소를 반환한다.
- `PDPE` : PD의 주소를 가리킴 
- `PDE` : PT의 주소를 가리킴
- `PTE` : Physical Frame의 주소를 가리킴 

---
### 1.3.  4-Level Paging 사용 이유 

- *배경 : 64bit CPU의 등장* 
	- CPU가 64bit 아키텍쳐로 전환되면서 프로세스가 사용할 수 있는 VA공간이 급격하게 증가 
	- 기존 32bit는 4GB까지만 주소 표현이 가능했지만, **64bit CPU는 훨씬 넓은 주소 공간이 필요**로 해졌다. ($2^{64 - 12}=256TB$)
	- 이 **거대한 주소 공간을 매핑하고 대규모 어플리케이션의 요구사항을 충족시키기 위해** 새로운 페이징 기법이 필수적 
	  
- 💢*기존 한계*
	1. *단일 테이블 한계(희소성)*
		- 단일 페이지 테이블 구조는 프로세스의 가상 주소 공간 전체를 커버하기 위해 모든 PTE를 **연속적으로 미리 생성**해야한다.
		- *메모리 낭비*
			- 프로세스가 256TB의 주소 공간 중 **실제로 사용하는 영역(코드, 힙, 스택)은 극히 일부분**이다. 
			- 단일 테이블을 사용하면 **사용하지 않는 주소 공간에 대한 엔트리까지 모두 생성**해야 하므로, 페이지 테이블 자체를 저장하는 데 엄청난 물리 메모리가 낭비된다.
		  
	2. *3-level paging 한계(공간 한계)* 
		- 기존 3-level paging 으로는 512GB만 커버가 가능했기에 많은 주소 공간들을 담지 못했다.
		  
- ✅*4-level-paging*
	1. *넓은 주소 공간 매핑*
		- 4-level-paging은 **훨씬 넓은 256TB(48bit)의 가상 주소 공간을 효율적으로 다룰 수 있게 설계**되어 있다.
		  
	2. *메모리 절약, 동적 확장* 
		- 4개의 레벨로 주소를 분할함으로써, **사용되지 않는 거대한 주소 범위**에 대해서는 최상위 레벨(PML4T, PDPT)의 엔트리만 'Not Present'로 표시하고, **하위 페이지 테이블을 아예 생성하지 않는다.**
	![Pasted image 20251013092924.png](/img/user/supporter/image/Pasted%20image%2020251013092924.png)


---
### 1.4.  CPU가 어떻게 PML4를 사용할까?

>[!QUESTION] 어떻게 CPU가 pml4를 사용할까❓
>- [[Computer_Science/Virtual_Memory/Virtual Memory\|Virtual Memory]]에서 말했듯이 각 프로세스(pintos에서는 thread)는 자신만의 독립된 페이지를 메모리에 보관한다. 
>- 즉, CPU가 Page Table을 활용하기 위해서는 이 메모리 주소가 필요한 것. 
>- 따라서, Process 전환 시마다 각 프로세스의 pml4 주소가 CPU레지스터에 등록되게 하여 MMU(Memory Management Unit)가 해당 프로세스의 가상 주소 공간을 사용하도록 한다
>- *pml4만 알면 물리 주소(pa)에 접근 가능*
>	- 가상 주소 ➡ pml4 ➡PDPE ➡PDE ➡PTE ➡ PA (물리 주소) 


CPU가 메모리에 접근할 때는 Page Table을 통한 주소변환(가상 주소 ➡ 물리 주소)으로 접근한다.
Lazy Loading전략을 취하는 OS에서는 페이지가 Load되지 않는 경우가 많고 해당 가상 주소에 접근할 경우  Page Table에서는 `PTE(Present bit)`가 0으로 표시되어 Page Fault Execption을 터트린다. 이 때 fault관련 정보들은 레지스터와 에러코드로 전달된다
```text
VA -> ... -> PTE = 0  ====> Page Fault 터트림 
```

아래는 **Pintos에서 `page_fault`부분을 다루는 함수**이다(기존에 세팅되어 있음)
```c
static void page_fault(struct intr_frame *f) {
  bool not_present; /* True: not-present page, false: writing r/o page. */
  bool write;       /* True: access was write, false: access was read. */
  bool user;        /* True: access by user, false: access by kernel. */
  void *fault_addr; /* Fault address. */


  /* Obtain faulting address, the virtual address that was
     accessed to cause the fault.  It may point to code or to
     data.  It is not necessarily the address of the instruction
     that caused the fault (that's f->rip). */

  fault_addr = (void *)rcr2();
  /* Turn interrupts back on (they were only off so that we could
     be assured of reading CR2 before it changed). */
  intr_enable();
  /* Determine cause. */
  not_present = (f->error_code & PF_P) == 0;
  write = (f->error_code & PF_W) != 0;
  user = (f->error_code & PF_U) != 0;
```


---
## 2.  Frame
```C
/* The representation of "frame" */
// 보통 4KB 크기의 물리 페이지 나타냄 
struct frame {
  void *kva; // 프레임의 커널 가상 주소 
  struct page *page; // 이 frame과 연결된 사용자 페이지 
};
```
---
## 3.  Page
```c
/* The representation of "page".
 * This is kind of "parent class", which has four "child class"es, which are
 * uninit_page, file_page, anon_page, and page cache (project4).
 * DO NOT REMOVE/MODIFY PREDEFINED MEMBER OF THIS STRUCTURE. */

// Pag

// SPT안에 메타데이터들을 담은 Page를 넣자
struct page {
  const struct page_operations *operations;  // 어떤 타입의 페이지인지 결정하는 함수 테이블
  void *va;              // 이 페이지가 속한 User 공간의 가상주소
  struct frame *frame;   // 매핑된 물리 Frame 
  /* Your implementation */
  // 페이지 타입, 백업 스토어 정보, ops함수 포인터, va 
	... 
  
  /* Per-type data are binded into the union.
   * Each function automatically detects the current union */
  union {
    struct uninit_page uninit;
    struct anon_page anon;
    struct file_page file;
#ifdef EFILESYS
    struct page_cache page_cache;
#endif
  };
};
```


## 4.  Page Table vs Supplemental Page Table(SPT)
>[!QUESTION] 궁금증
>왜 OS는 Page Table과 Supplemental Page Table을 따로 두는가?

### 4.1.  Preview - 간단 차이 비교
- Page Table: 하드웨어가 사용하는 매핑 (VA→PA).
- SPT: OS가 관리하는 보조 구조 (페이지 상태/위치 관리).

>[!tip] 주요 차이 - 담고 있는 정보 
>1. *Page Table - CPU용 주소 변환기*
>	- CPU 하드웨어(MMU)가 메모리 접근 시 직접적으로 참고하는 곳 
>	- **하드웨어가 강제하는 최소 정보(매핑 정보)** 만 담음 
>		- Present Bit : 물리 Frame이 실제로 존재하는지
>		- Read/Write bit, Dirty/Accessed Bit 등 
>	- 즉, 단순히 '**지금 메모리에 어떤 페이지가 올라와 있는지?**' 확인 용 
>	- 한계(부족한 점)
>		- 없는 페이지에 대한 정보가 없음 
>		  
>2. *SPT - 페이지 메타데이터 저장소* 
>	- **올라오지 않은 페이지를 포함한 전체 가상 메모리의 청사진.**
>	- 단순 **Page Table은 현재 로드되지 않은 페이지에 대한 정보가 없다.** 따라서 OS는 **지연 로딩, 스왑, mmap, 스택 자동성장** 같은 기능을 위해 **페이지의 출처/복귀 경로**를 별도로 관리해야 함
>	- 이걸 관리하는 곳이 Supplemental Page Table이다.

---
### 4.2.  Page Table
CPU/MMU가 쓰는 H/W용 

---
#### 4.2.1.  페이지 테이블 쓰는 이유 in VM
1. *효율적 메모리 사용* : **실제로 사용하는 페이지에 대해서만** Frame 할당
2. *보호* : 접근 권한 제어를 통해 **메모리 접근 방지** 가능
3. *유연성* : 여러 프로세스가 서로 다른 주소 공간에 안전하게 접근 가능
4. *확정성* : **계층적** 페이지 테이블을 통해 **메모리 오버헤드 없이** 거대한 주소 공간을 처리

---
#### 4.2.2.  PTE
Page Table은 여러 개의 Page Table Entry(PTE)로 구성되어 있다.
> PTE는 가상 ↔ 물리 주소 변환에 필요한 저수준 메타데이터가 들어감

참고 자료 : [GeeksForGeeks의 OS PTE](https://www.geeksforgeeks.org/operating-systems/page-table-entries-in-page-table/)

![Pasted image 20250926233112.png](/img/user/supporter/image/Pasted%20image%2020250926233112.png)
Page Table의 각 매핑은 Page Table Entry에 저장된다. PTE에 필수 메타데이터를 저장하여 시스템이 메모리를 안전하고 효율적으로 관리할 수 있도록 함

✔*PTE에 저장되는 정보*
1. *Frame Number*
	- **페이지가 저장된 물리 메모리의 Frame을 식별**
	- 필요한 비트 수 =  $log_2​(물리 메모리 크기 / 프레임 크기)$
	  
2. *Present/Absent Bit (or Valid/Invalid Bit)*
3. *Protection Bits(보호 비트)*
4. *Referenced Bit(참조 비트)*
5. *Caching*
	- 해당 페이지가 캐시되어야 하는지 여부 결정
	- 항상 최신 데이터를 얻어야 하는 입출력에 대해서는 캐싱을 비활성화한다.
	  
6. *Dirty Bit(Modified Bit)*
	- 페이지가 쓰기 작업에 의해 수정되었는지 여부를 나타냄
	- 수정 됨(1) : 페이지가 메모리에서 Evicted 될 때 Disk에 다시 기록 必
	- 수정 X(0) : 페이지를 디스크에 다시 기록할 필요 없이 폐기 가능


> [!INFO] VM Entry
> 각 가상 메모리 페이지에 대한 정보를 담고 있는 항목
> - 특정 가상 페이지가 어떤 물리 프레임에 매핑되어 있는지
> - 스왑 영역에 있는지 다른 영역에 있는지 등 


---
### 4.3.  SPT
#OS전용_테이블 

SPT = Supplemental Page Table

> [!WARNING] 단순 Page Table의 한계 
> 단순 **Page Table은 현재 로드되지 않은 페이지에 대한 정보가 없다.**
> 즉,  OS는 **지연 로딩, 스왑, mmap, 스택 자동성장** 같은 기능을 위해 **페이지의 출처/복귀 경로**를 별도로 관리해야 함 -> 그게 바로 SPT


#### 4.3.1.  개념 
> 올라오지 않은 페이지를 포함한 전체 가상 메모리의 청사진.

- 목표 : 어떤 가상 주소에 대응되는 VM Entry를 효율적으로 찾을 수 있게 하는 것
- 로딩/스왑관리/파일 매핑 등 **고수준의 VM관리에 필요한 데이터를 관리**하는 테이블
- **각 프로세스마다 독립적**으로 유지됨
- 커널(OS)이 관리한다(H/W가 아닌) 
- 보통 Hash 기반

#### 4.3.2.  어떻게 활용 가능한가❓⭐
> 보통 SPT의 메타 데이터는 Page Fault 발생 시 OS가 처리할 때 사용 

1. 해당 가상 주소의 **메타데이터를 기록**
	- Pintos에서는 `vm_alloc_page_with_initializer`라는 함수를 통해 **해당 가상주소에 해당하는 page 메타데이터를 등록**한다.
	```c
	bool vm_alloc_page_with_initializer(enum vm_type type, void* upage, bool writable, vm_initializer* init, void* aux) {

    struct supplemental_page_table* spt = &thread_current()->spt;
	/* Check wheter the upage is already occupied or not. */
	if (spt_find_page(spt, upage) == NULL) {
      struct page* page = malloc(sizeof(struct page));
	    .....
	```
	  
2. *Page Fault 처리 시 참조* ⭐
	- Lazy Load시 Page를 미리 Load하지 않아 VA를 통한 접근시 Page Fault가 발생할 것이다.
	- 이 때, Page Fault난 가상 주소의 **VM Entry 를 SPT에서 찾고,**
	- **디스크에서 로드**하거나 **스왑영역에서 복원**(메모리에 없는 영역도 추적)
	- 조건은 따지기 必
		- 단순히 번호 매핑이 아니라, 개별 페이지에 대한 **더 많은 정보가 필요**하다(ex. 디스크 위치, `file-backed`인지 메모리에만 있는지 등의 유형)
		- *페이지, offset 번호*
		- *읽기/쓰기 권한*
		- *가상 페이지의 유형*
			- 실행 파일의 페이지인지
			- 일반 파일의 일부인지
			- 스왑 영역의 일부인지 -> 익명 페이지 

>[!QUESTION] 실행(executable) 파일의 페이지인지 아닌지 구별하는 이유
>- 실행 파일이 수정되는 것을 금지하기 위해 
>- 실행 파일을 실행할 때 읽을 수 없고 수정할 수 없도록 만들 수 있음 

#### 4.3.3.  실제 OS에서 SPT
> [!INFO] 실제 OS(Linux 등)에서 SPT?
> - 실제 OS에서는 SPT와 비슷한 역할을 하는 자료구조는 있는데 이름도 다르고 구현은 훨씬 복잡
> - 커널이 **여러 구조체(mm_struct, vm_area_struct 등)** 를 기반으로 전체 가상 메모리 공간을 관리


### 4.4.  궁금증 : PTE에 메타데이터가 있는데 SPT가 왜 필요❓

**더 많은 메타데이터 필요**
- 막상 VM관리할 때 필요한 정보는 PTE의 정보 말고도 훨씬 많다.
- PTE에 있는 메타데이터는 단순히 H/W수준에서의 필요 정보들이고 OS 수준으로가면 더 필요. 
- 이를 SPT에 넣는 것

![Pasted image 20250927101453.png](/img/user/supporter/image/Pasted%20image%2020250927101453.png)

> 보통 SPT의 메타 데이터는 Page Fault 발생 시 OS가 처리할 때 사용 

### 4.5.  VM 핵심 - Page Fault 처리 흐름
참고 : [[Pintos/Pintos VM/Pintos VM - 구조체 및 관련 개념 정리\|Pintos VM - 구조체 및 관련 개념 정리]] -> Page Table vs SPT

1. CPU가 VA 접근 → Page Table에서 **P=0** 또는 권한 위반 → **페이지 폴트** 발생
2. 커널이 **SPT 조회**
    - 항목 없음 → **정당하지 않은 접근** ⇒ 프로세스 종료
    - 항목 있음 → 출처 판별(실행파일/스왑/mmap 등)
        
3. **프레임 확보**(프레임 테이블·교체정책 사용) → **해당 출처에서 Page Load**
4. **Page Table에 매핑 설치**(P=1, 권한 비트 세팅), TLB 갱신
5. 필요 시 SPT 메타데이터 업데이트(예: 스왑 슬롯 해제 등)


## 5.  Page 타입과 타입별 특징

### 5.1.  익명 페이지
> 어떤 파일에도 매핑되지 않은 페이지 (**파일 → 메모리 매핑** 관계가 없는 메모리 공간)


#### 5.1.1.  특징
1. *파일 매핑이 없음*
	- 디스크와의 파일 연결 필요 없음
	  
2. *초기화 방식 - 0으로*
	- 생성될 때 **보통 0으로 초기화된 페이지가 할당**된다.
	- **목적 = 보안** : 프로세스 간의 데이터 유출을 막기 위해
	  
3. *스왑(Swap) 가능* Using **swap disk**
	- 메모리 부족으로 인해 익명 Page가 쫓겨난다면 그냥 Swap 공간으로 가면 됨(Swap-Out)
	- 필요하면 Swap공간에서 복원하면 되고(Swap-In)
	  
>[!QUESTION] 스왑 공간이란 ❓
> OS가 가상 메모리를 구현하기 위해 사용하는 2차 메모리의 특별한 공간 
> (RAM 부족 시 비상 공간)

#### 5.1.2.  주 사용처
> 실행 중 **동적으로 관리되는 메모리**애서 자주 사용된다.

1. *힙 영역* 
	- `malloc()`내부에서 `sbrk`등으로 **새롭게 메모리 생성 시 사용**됨
	  
2. *스택 영역*
	- 함수 호출 시 **자동으로 확장되는 공간**에서 사용됨


### 5.2.  file-backed 페이지
#힙 #스택 

#### 5.2.1.  개념 
> 디스크에 있는 파일 내용을 그대로 메모리에 매핑해 사용하는 페이지

- VM의 특정 페이지 - 디스크 상의 특정 파일  << 연결 
- 익명 페이지와 달리 스왑 공간에 연결되지 않는다.

#### 5.2.2.  특징
1. *원본이 존재*
	- 디스크에 항상 해당 파일의 원본 데이터가 있음.
	- 따라서 메모리 부족 시 **페이지를 그냥 버릴 수 있음** (dirty 상태가 아니면).
	  
2. *공유가 가능* - 메모리 절약 핵심 
	- **동일한 파일**을 **여러 프로세스가 매핑**할 수 있다. 
	- 이때 각 프로세스는 **같은 물리 Frame을 공유**
	  
3. *수정 여부에 따른 처리* ⭐
	1. **수정 안 된 상태**
		- 메모리에서 쫓겨나면 그냥 버림 ⬅스왑 영역/비용 절약
		- 필요하면 다시 디스크에서 읽음
		  
	2. **수정된 상태(Dirty)**
		- OS는 **변경된 내용을 원본 파일의 해당 위치에 다시 써야** 함(I/O작업)
		- EX. 파일 매핑(`mmap`)이면 디스크 파일에 write
		  
4. *보안 측면*
	- 보통 실행 파일 코드 영역을 **읽기 전용**으로 매핑해서, 프로세스가 잘못 덮어쓰지 못하도록 보호.

#### 5.2.3.  주 사용처 
![Pasted image 20250927151636.png](/img/user/supporter/image/Pasted%20image%2020250927151636.png)
1. *실행 파일 영역 매핑* - 코드 영역, 데이터 영역
	- **가상 메모리 주소 영역과 디스크 상의 실행 파일 내용이 논리적으로 연결**
	- **Lazy Loading** 
		- 메모리에 파일 데이터를 한 번에 올리지 않고 **가상 주소에 매핑만** 해둠 
		- 이로 인해, 메모리 절약 및 빠른 시작 가능 
		  
	- **사용 영역** : 코드 영역(`.text 영역`), 라이브러리 영역
		- 읽기 전용으로 매핑 
		  
	- 만약, 특정 코드가 필요하면 OS가 페이지 단위로 파일 내용을 메모리에 올린다.
	- `exec()` 시스템 콜에서 실행 파일 로딩 시 
	  
| 가상 주소 (Virtual Address) | 물리 주소 (Physical Address) | 연결된 파일 위치 (File Offset) | 권한 (Permission) |     |
| ----------------------- | ------------------------ | ----------------------- | --------------- | --- |
| `0x100000` ~ `0x100FFF` | **(비어있음/Invalid)**       | 실행 파일의 0KB ~ 4KB 위치     | 읽기/실행           |     |
	  
2. *메모리 매핑 I/O(`mmap`)*
	- 파일을 open하고 `mmap()`으로 특정 주소 범위에 매핑 시 파일의 내용을 마치 배열처럼 접근 가능
	- 장점 : 시스템 콜 오버헤드 줄이고 곧바로 유저 공간에서 접근 가능(버퍼 관리 필요 없음)



## 6.  Page_Operations
> 페이지가 어떤 행동을 할 수 있는지 정의한 함수 모음집

C에서는 클래스도 없고 구조체 내의 메서드 개념이 없다. 대신, **함수 포인터를 구조체 내에 넣어서 메서드처럼 씀(흉내)**

페이지의 타입에 따라 `page_operations` 테이블이 연결됨

아래는 Pintos에서 정의된 Page 종류에 따른 `page_operations` 테이블들이다.
![Pasted image 20250927103803.png](/img/user/supporter/image/Pasted%20image%2020250927103803.png)


