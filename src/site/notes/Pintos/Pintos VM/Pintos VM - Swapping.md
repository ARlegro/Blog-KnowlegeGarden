---
{"dg-publish":true,"permalink":"/Pintos/Pintos VM/Pintos VM - Swapping/","noteIcon":"","created":"2025-10-13T15:20:26.854+09:00","updated":"2025-10-15T16:25:03.792+09:00"}
---




> 물리 메모리의 활용을 극대화하기 위한 **메모리 회수기법**

관련 개념
- [[Pintos/Pintos VM/Pintos VM - Bitmap 자료구조 공부\|Pintos VM - Bitmap 자료구조 공부]]

### 0.1.  목차

[[#들어가기 전 (개념, 자료 분석)|들어가기 전 (개념, 자료 분석)]]
- [[#들어가기 전 (개념, 자료 분석)#1.  메모리 관점에서 swapping이란?|1.  메모리 관점에서 swapping이란?]]
- [[#들어가기 전 (개념, 자료 분석)#2.  GitBook 내용들|2.  GitBook 내용들]]
	- [[#2.  GitBook 내용들#2.1.  익명 페이지와 스왑 - GitBook|2.1.  익명 페이지와 스왑 - GitBook]]
	- [[#2.  GitBook 내용들#2.2.  스왑테이블|2.2.  스왑테이블]]
- [[#들어가기 전 (개념, 자료 분석)#3.  관련 용어 - slot, sector|3.  관련 용어 - slot, sector]]

[[#Swap_Init 구현|Swap_Init 구현]]

[[#익명 페이지 Swapping 구현|익명 페이지 Swapping 구현]]
- [[#익명 페이지 Swapping 구현#1.  익명 페이지 Swap_In()|1.  익명 페이지 Swap_In()]]
	- [[#1.  익명 페이지 Swap_In()#1.1.  분석 및 구현 추리|1.1.  분석 및 구현 추리]]
	- [[#1.  익명 페이지 Swap_In()#1.2.  구현|1.2.  구현]]
- [[#익명 페이지 Swapping 구현#2.  익명 페이지 Swap Out|2.  익명 페이지 Swap Out]]

 [[#File-Backed 페이지 Swapping|File-Backed 페이지 Swapping]]
- [[#File-Backed 페이지 Swapping#1.  File-Backed 페이지 Swap In|1.  File-Backed 페이지 Swap In]]
- [[#File-Backed 페이지 Swapping#2.  File-Backed 페이지 Swap Out|2.  File-Backed 페이지 Swap Out]]

---
# 들어가기 전 (개념, 자료 분석)

---
## 1.  메모리 관점에서 swapping이란?

> **물리 메모리가 부족**할 때, 일시적으로 **메모리의 일부 페이지를 보조기억장치(디스크)로 옮겨서 사용 가능한 메모리를 확보하는 기법**

*한정된 메모리를 효율적으로 활용하는 방식* 
- 사용하지 않는 페이지를 swap_disk 영역으로 내보내고 
- 필요한 시점에 다시 메모리로 불러오는 방식

*동작 단계*
1. *메모리 부족 발생*
	- 새로운 페이지를 Load할 공간이 없을 때 발생하는 문제 
	  
2. *Swap Out*
	- 새로운 페이지를 Load할 공간을 확보하기 위해 적절한 알고리즘에 의해 특정 페이지를 메모리에서 디스크로 내보낸다(Evict)
	  
3. *Swap In*
	- 2번에서 내보냈던 특정 페이지가 필요해지면 Disk에서 메모리로 복원 


>[!tip] Swapping은 메모리 확장의 핵심 가상화 기술


> [!WARNING] Swap은 file-backed page는 필요 없다
> - file-backed page는 어차피 원본은 disk에 있다. 근데 이걸 swap_disk 영역을 따로 만들어서 또 disk에 보관하는 불필요한 동작을 할 필요가 없다.
> - 따라서 **Swap기법은 익명 페이지를 대상**으로 하면 될 것이다.

---
## 2.  GitBook 내용들
`swap_out` 된 페이지에 접근하려고 할 때, `OS`는 `disk`에 복사해둔 내용을 그대로 다시 `RAM` 메모리에 가져옴으로써 페이지를 다시 복원시킴 

`swap_in/out` 연산들은 함수 포인터에 의해 호출된다. 이 함수 포인터는 `page_operations`에 등록되어 있음 


---
### 2.1.  익명 페이지와 스왑 - GitBook
`vm/anon.c`에서 `vm_anon_init` 및 `anon_initializer`를 수정합니다. 
- 익명 페이지에는 백업 저장소가 없습니다. 
- 익명 페이지의 스와핑을 지원하기 위해 **swap_disk라는 임시 백업 저장소를 제공**합니다. 
- `swap_disk`를 활용하여 익명 페이지에 대한 스왑을 구현합니다.


---
### 2.2.  스왑테이블 
[[Pintos Project3 - 스왑테이블 구현\|Pintos Project3 - 스왑테이블 구현]]
스왑 테이블은 **사용중인 스왑 슬롯**과 **빈 스왑 슬롯들을 추적**합니다. 
- 프레임에 있는 페이지를 스왑 파티션으로 쫓아내기 위해서, 스왑 테이블은 미사용된 스왑 슬롯을 고를 수 있도록 해줘야 합니다. 
- 페이지가 다시 읽혀서 돌아가거나, 페이지 주인인 프로세스가 종료되어 버릴 경우에는 스왑 테이블이 스왑 슬롯을 free 해줄 수도 있어야 합니다.

스왑 슬롯은 느긋하게 할당되어야 합니다. 이 말은, eviction에 실제로 필요할 때만 할당되어야 한다는 말입니다. 프로세스가 시작될 때 실행파일에서 데이터 페이지들을 읽고 스왑에 곧바로 쓰는 행위는 느긋하지 못한 행위입니다. 특정 페이지를 저장하기 위해 스왑 슬롯이 예약되어서는 안 됩니다.

스왑 슬롯의 내용물이 프레임으로 읽혀 돌아오면 그 때 스왑 슬롯을 free 해주면 됩니다.


## 3.  관련 용어 - slot, sector 

1. *slot - 페이지 저장하는 공간 (논리적 개념)*
	- *개념* : OS가 메모리 관점에서 사용하는 논리적 개념의 저장 단위 
	- Swap 영역에서 **하나의 메모리 페이지를 저장하기 위해** 할당되는 공간 
	- *왜❓* 
		- Swapping과정에서 디스크상의 저장 위치를 추상적으로 표현 가능
	- *크기* : 페이지 크기와 동일하게 한다.(4KB)
	- **하나의 slot는 여러 개의 sector로 구성된다.**
	  
2. *sector* 
	- **물리적인 저장 장치가 데이터를 저장**하는 가장 기본적인 **물리적 단위** 
	- 크기 : 보통 512 byte(최신은 좀 다르다고 하네)
	- **디스크 I/O가 발생하는 최소 단위**이다 ⭐
		- 이로 인해 디스크 write 시 sector단위로 I/O 접근 必

---

# Swap_Init 구현 
`Swap`동작을 구현하기 위해서는 몇 가지 세팅을 해야 한다.
- `swap_disk` 생성 
- 자료 구조 - `bitmap`
- `swap`용 `lock` : `swap`도 `disk`에 접근하는 것이기 때문

```C
#define DISK_SECTOR_SIZE 512
#define PGSIZE (1 << 12) // 4096
#define SECTORS_PER_PAGE (PGSIZE / DISK_SECTOR_SIZE) // 8 

static struct disk *swap_disk;
static struct bitmap *swap_bitmap;
static struct lock swap_lock;
static size_t max_bitmap_idx;
```

```c
void vm_anon_init(void) {
	// swap_disk 생성 
  swap_disk = disk_get(1, 1);
  disk_sector_t total_sectors = disk_size(swap_disk);
  size_t total_slots = total_sectors / SECTORS_PER_PAGE; // total slot 계산
  
  // bitmap 자료구조 생성 
  swap_bitmap = bitmap_create(total_slots);
  max_bitmap_idx = total_slots - 1;
  
  // swap용 lock_init()
  lock_init(&swap_lock);
}
```
bitmap.c에 있는 스켈레톤 코드를 사용하면서 init을 구현했다.

`disk` 생성 
- `disk_get` 스켈레톤 코드를 활용
- 1, 1은 스켈레톤 코드에서 swap 용 disk를 할당받고 싶을 때 쓰는 인자
	```c
	/* Returns the disk numbered DEV_NO--either 0 or 1 for master or
	   slave, respectively--within the channel numbered CHAN_NO.
	   Pintos uses disks this way:
	0:0 - boot loader, command line args, and operating system kernel
	0:1 - file system
	1:0 - scratch
	1:1 - swap
	*/
	struct disk *disk_get (int chan_no, int dev_no) {
	```

*`total slots` 계산* 
- disk_size는 해당 disk에 몇 개의 sector들이 있는지 확인할 수 있다.
- 하나의 slot은 Page의 사이즈와 같게 설정하니까 slot개수 = 총 total_setor / 8 이다.
	- 한 Slot 크기 = 한 페이지 크기 (보통 4KB)
	- 하나의 Slot에 있는 sector 수 (논리적) = $Slot크기 / 512 bytes$
![Pasted image 20251015104805.png](/img/user/supporter/image/Pasted%20image%2020251015104805.png)
(참고 : [토마스 KRENN](https://www.thomas-krenn.com/en/wiki/Advanced_Sector_Format_of_Block_Devices) )

`bitmap`자료구조 생성

`swap`용 `lock_init`

(Bitmap 개념 참고 : [[Pintos/Pintos VM/Pintos VM - Bitmap 자료구조 공부\|Pintos VM - Bitmap 자료구조 공부]])

# 익명 페이지 Swapping 구현 

---
## 1.  익명 페이지 Swap_In()

> 익명 페이지가 swap_disk 영역에서 다시 메모리로 Load되는 과정과 관련된 함수

일단 이전에 익명 페이지 임시 저장소용 `disk`를 `init`으로 준비했다.
이를 활용해서 swap_in을 하면 될 것 같은데...

---
### 1.1.  분석 및 구현 추리
그럼 disk를 읽는?그런 스켈레톤 코드가 있는지 확인해보면 될 것 같다.
`disk.c`에 들어가면 `disk_read`라는 함수가 있다.
```c
/* Reads sector SEC_NO from disk D into BUFFER, which must have
   room for DISK_SECTOR_SIZE bytes.
   Internally synchronizes accesses to disks, so external
   per-disk locking is unneeded. */
void disk_read (struct disk *d, disk_sector_t sec_no, void *buffer) {
```
근데 `disk_secotr_t?` 이걸 어케해야할까❓
`page`구조체에 새로 필드를 넣어야 하나❓
맞는 것 같다. GitBook을 다시 보니 
```c
bool anon_initializer (struct page *page, enum vm_type type, void *kva);
```
```text
이것은 익명 페이지의 이니셜라이저입니다. 
Swapping을 지원하려면 anon_page에 몇 가지 정보를 추가해야 합니다.

이제 vm/anon.c에서 anon_swap_in 및 anon_swap_out을 구현하여 익명 페이지에 대한 스와핑을 지원합니다. 페이지를 swap in하려면 페이지를 swap out해야 하므로 anon_swap_in을 구현하기 전에anon_swap_out을 구현하는 것이 좋습니다. 데이터 내용을 스왑 디스크로 이동하고 안전하게 메모리로 다시 가져와야 합니다.
```
라고 되어있다.

*✅생각 🤔*
- 일단 annon_page에 disk_sector_t 정보를 넣으면 될 것 같고, 내용을 보면 swap in보다는 swap out을 먼저해야 하나 싶네....
- 그래도 테스트 돌리기 전에 둘 다 구현하고 돌릴거면 상관은 없을 듯?
```c
// 초기 anon_page 구조체 
struct anon_page {

};
```

`disk_sector_t`에 대한 정보를 조사하다가 아래와 같은 주석을 봤다.
```c
/* Index of a disk sector within a disk.
 * Good enough for disks up to 2 TB. */
typedef uint32_t disk_sector_t;
```
- swap_disk는 slot 단위로 쪼개져 관리되는데
- 한 slot당 크기는 메모리의 한 페이지와 동일

---
### 1.2.  구현 
앞서 분석한 내용과 내 추리에 의존해서 코드를 작성했다.

큰 그림은 아래와 같다.
1. 디스크에 저장된 페이지 데이터 가져오며 물리 Frame 복원하기
2. 페이지 테이블(pml4)에 다시 매핑하기 


```c
/* Swap in the page by read contents from the swap disk. */

static bool anon_swap_in(struct page *page, void *kva) {

  struct anon_page *anon_page = &page->anon;
  
  disk_sector_t sector_no = NULL;
  // 1. Page 슬롯 정보 확인 - 해당 페이지가 어디 slot에 있는지 확인 
  size_t slot_idx = anon_page->slot_idx;
  lock_acquire(&swap_lock);
  // 슬롯 idx 범위 검증
  if (slot_idx > max_bitmap_idx) {
    return false;
  }
  
  // 이건 확인용 - 스켈레톤 코드에 있길래 한번 써봤다.
  if (!bitmap_test(swap_bitmap, slot_idx)) return false;

	// Disk에서 Frame으로 복사 : slot 전체 읽기(읽는 단위는 SECTOR)
  for (int i = 0; i < SECTORS_PER_PAGE; i++) {
    sector_no = (slot_idx * SECTORS_PER_PAGE) + i;
    disk_read(swap_disk, sector_no, kva + (i * DISK_SECTOR_SIZE));
  }
  // Bitmap 갱신
  bitmap_set(swap_bitmap, slot_idx, false);
  lock_release(&swap_lock);

	// 페이지 테이블(pml4) 매핑 복원
  pml4_set_page(thread_current()->pml4, page->va, page->frame->kva, true);
  anon_page->slot_idx = SIZE_MAX;
  return true;
}
```

*동작 흐름*
1. *페이지 및 슬롯 정보 확인* 
	- 해당 페이지가 디스크 어디에 저장되어 있는지(`slot_idx`)를 가져온다.
	- `slot_idx` : swap영역 내의 slot 번호 
	  
2. *Lock 획득 및 검증*
3. *디스크에서 Frame으로 데이터 복사*
	- slot 전체 읽기(읽는 단위는 SECTOR)
	- 한 페이지는 한 Slot크기로 저장되어 있고 한 Slot은 여러 Sector로 이루어져 있다.
	- 또한, **Disk I/O 작업은 보통 Sector단위**로 이루어지기에 for문으로 돌기
	  
4. *Bitmap 갱신*
	- swap 영역에서 해당 슬롯이 이제 비게 되었으므로, `swap_bitmap`의 해당 비트를 `false`로 변경한다.  
	- 이후 이 slot은 다른 페이지가 재활용할 수 있다.
    
5. *락 해제*
	 - 작업이 끝났으므로 락 해제.
	   
6. *페이지 테이블(pml4) 매핑 복원*
	- 새로 로드된 프레임(`page->frame->kva`)을 해당 페이지의 가상주소(`page->va`)에 매핑한다.
	- `slot_idx`는 더 이상 유효하지 않으므로 `SIZE_MAX`로 초기화.


---
## 2.  익명 페이지 Swap Out

아래 코드는 익명 페이지를 물리 메모리에서 Evict할 때 디스크의 swap영역으로 내보내는 swap-out 루틴이다.

큰 그림을 먼저 말하자면
1. 빈 Slot 탐색 및 할당 - `bitmap_scan_and_flip`
2. Evict할 페이지 내용을 slot에 기록 - `swap_disk`에 보관
3. PTE해제
4. page에 저장된 slot 위치 적기 

```c
/* Swap out the page by writing contents to the swap disk. */
/**
 * 현재 frame 내용을 swap space로 보내기(1페이지 8섹터 기록)
 * PTE 해제 + frame 반환? or 분리?
 * 페이지는 메모리가 아닌 swap slot보관 상태로 전환
 */
static bool anon_swap_out(struct page *page) {
  struct anon_page *anon_page = &page->anon;
  struct frame *frame = page->frame;
  if (frame == NULL) return false;

  void *kva = frame->kva;
  lock_acquire(&swap_lock);
  // 빈 슬롯 할당
  size_t slot_idx = bitmap_scan_and_flip(swap_bitmap, 0, 1, false);  // false 는 빈거
  
  // 가용 슬롯 있는지 확인 : BITMAP_ERROR면 가용 슬롯 없음
  if (slot_idx == BITMAP_ERROR) return false;

	// 디스크에 기록 : 페이지 단위 → 섹터 단위로 디스크 기록
  for (int i = 0; i < SECTORS_PER_PAGE; i++) {
    disk_sector_t sector = (slot_idx * SECTORS_PER_PAGE) + i;
    disk_write(swap_disk, sector, kva + (i * DISK_SECTOR_SIZE));
  }
  lock_release(&swap_lock);

  pml4_clear_page(frame->owner_pml4, page->va);
  anon_page->slot_idx = slot_idx;
  return true;
}
```
*동작 흐름*
1. *빈 슬롯 탐색 및 할당*
	- 비어 있는(false) 비트를 찾아 **즉시 true로 뒤집어 예약**”하므로, 다른 스레드가 같은 슬롯을 가져가는 **Race Condition을 원천 차단**한다
   
2. *디스크에 기록*
	- **슬롯의 시작 섹터**부터 연속으로 페이지 전체를 기록.
	  
3. *PTE 매핑 해제*
	- 이제 해당 가상 주소에 접근하면 Page Fault가 난다(이 때, swap_in()이 일어남)
	  
4. *메타데이터 등록*
	- 이 페이지가 어디 swap_slot에 존재하는지 표시
	- 이는 swap_in 과정에서 필요

---
# File-Backed 페이지 Swapping

>[!QUESTION] 익명 페이지랑 뭐가 다른가❓
>- *익명 페이지* : swap 공간으로 내보내고, `swap_in` 시 해당 슬롯에서 복구.
>- *파일 페이지*
>	- swap을 거치지 않음 ⭐
>	- Dirty면 **원본 파일에 write-back**만 하면 되고, 
>	- Clean이면 단순 `unmap`. 
>	- 다음 접근 때 **파일에서 다시 lazy-load** 하면 끝.

---
## 1.  File-Backed 페이지 Swap In
> File-backed page를 메모리로 복구하는 루틴 

별거 없다. 단순히 File에서 read_byte하고 나머지는 0으로 채우기
익명 페이지와 달리 swap_disk, slot을 사용하지 않고 바로 원본 file에서 복구하면 끝~

```c
/* Swap in the page by read contents from the file. */
// file swapin은 파일 read - 이거 그냥 copy_loaded_page로직 빌려쓰면될듯?
static bool file_backed_swap_in(struct page* page, void* kva) {
  enum vm_type type = VM_TYPE(page->operations->type);
  struct file_page* file_page = &page->file;
  size_t page_read_bytes = file_page->page_read_bytes;
  size_t page_zero_bytes = file_page->page_zero_bytes;

	// 파일 읽고
  off_t read_bytes = file_read_at(file_page->file, kva, page_read_bytes, file_page->ofs);
  ASSERT(read_bytes == page_read_bytes);
  // 페이지 나머지 부분 0으로 초기화화
  memset(kva + read_bytes, 0, page_zero_bytes);
  return true;
}
```

---
## 2.  File-Backed 페이지 Swap Out

> [!INFO] 로직 큰 그림 
> 1. 메모리에 올라와 있는 파일 기반 페이지를 내리고, 
> 2. 필요 시 수정분을 디스크 파일에 반영한 뒤, 
> 3. 현재 프로세스의 페이지 테이블에서 매핑을 제거


```c
static bool file_backed_swap_out(struct page* page) {
  struct file_page* file_page = &page->file;
  struct frame* frame = page->frame;
  void* va = page->va;
  if (frame == NULL) return false;

	// 올바른 pml4 선택
  uint64_t* owner_pml4 = frame->owner_pml4;

	// Dirty 여부 검사 및 쓰기반영(Write-back)
  if (pml4_is_dirty(owner_pml4, va)) {
    if (!dirty_writeback(page)) {
      return false;
    }
    // Dirty bit를 0으로 클리어 ✅
    pml4_set_dirty(owner_pml4, va, false);
  }

	// Page mapping 해제 
  pml4_clear_page(frame->owner_pml4, va);
  // 참조 관계 끊기
  frame->page = NULL;
  page->frame = NULL;
  return true;
}

static bool dirty_writeback(struct page* page) {
  struct file_page* file_page = &page->file;
  lock_acquire(&file_lock);
  off_t written_bytes = file_write_at(file_page->file, page->frame->kva, file_page->page_read_bytes, file_page->ofs);
  
  lock_release(&file_lock);
  return written_bytes == file_page->page_read_bytes;
}
```
*동작 흐름*
1. *올바른 `pml4` 선택* ⭐
	- 현재 실행하는 `thread`와 `swap_out` 할 페이지가 매핑된 `pml4`를 가진 `thread`가 다를 수 있다.
	- Swap_Out에 넘어온 Page와 그 안의 Frame은 **시스템 전역 Frame Table**에서 임의의 프레임을 축출한다 ➡ 그 Frame의 주체는 **현재 실행 스레드가 아닐 수 있다**.
	- `Dirty`/`Present`/`Accessed` 비트 등의 판단과 수정은 **그 프레임을 매핑한 PML4**에 대해 이뤄져야 정확하다.
	- 따라서 해당 `page`에 맞는 올바른 `pml4`를 선택하는 것이 중요
	  
2. *Dirty 여부 검사 및 쓰기반영(Write-back)*
	- Dirty bit를 0으로 클리어 必 : GitBook에 보면 CPU에 Dirty 감지는 해주는 데 0으로 초기화하는 것은 OS에서 하라고 했다.
	  
3. *페이지 매핑 해제* - `pml4_clear_page()`
4. *참조 관계 끊기*
	- 프레임과 페이지의 **양방향 연결을 해제**한다.
