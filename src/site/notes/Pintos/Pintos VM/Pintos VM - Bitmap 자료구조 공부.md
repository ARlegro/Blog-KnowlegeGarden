---
{"dg-publish":true,"permalink":"/Pintos/Pintos VM/Pintos VM - Bitmap 자료구조 공부/","noteIcon":"","created":"2025-10-08T11:46:53.199+09:00","updated":"2025-10-15T16:22:38.732+09:00"}
---




Pintos에서 swapping과정에서 사용할 자료구조인 bitmap 방식에 대한 공부 

### 0.1.  목차

- [[#개요|개요]]
	- [[#개요#1.  개념|1.  개념]]
	- [[#개요#2.  관련 용어 - slot, sector|2.  관련 용어 - slot, sector]]
	- [[#개요#3.  장점|3.  장점]]
		- [[#3.  장점#3.1.  장점 - swap 영역 관리 최적화|3.1.  장점 - swap 영역 관리 최적화]]
		- [[#3.  장점#3.2.  장점 - 공간 절약 (vs 정수형 배열)|3.2.  장점 - 공간 절약 (vs 정수형 배열)]]
		- [[#3.  장점#3.3.  빠른 검색|3.3.  빠른 검색]]
		- [[#3.  장점#3.4.  캐시 히트 향상|3.4.  캐시 히트 향상]]
- [[#사용|사용]]
	- [[#사용#1.  사용하는 스켈레톤 코드|1.  사용하는 스켈레톤 코드]]
		- [[#1.  사용하는 스켈레톤 코드#1.1.  create|1.1.  create]]
		- [[#1.  사용하는 스켈레톤 코드#1.2.  set|1.2.  set]]
		- [[#1.  사용하는 스켈레톤 코드#1.3.  bitmap_scan_and_flip|1.3.  bitmap_scan_and_flip]]
	- [[#사용#2.  swap disk에 사용하기|2.  swap disk에 사용하기]]

# 개요 

---
## 1.  개념 

> `bitmap`은 메모리나 디스크 공간의 **사용 여부를 효율적으로 관리**하기 위한 자료구조이다.

각 비트가 특정 자원(ex. 페이지, 블록, 슬롯 등)의 상태를 나타내며, **1비트당 하나의 자원**을 관리할 수 있어 매우 효율적 
- **1** → 사용 중    
- **0** → 비어 있음

>[!QUESTION] Pintos 어디서 이게 쓰일건데?
>`swap_disk` 구현 시에 **어떤 슬롯(섹터)이 비어 있는지, 사용 중인지**를 표시하는 데 사용

---
## 2.  관련 용어 - slot, sector 

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
## 3.  장점 

---
### 3.1.  장점 - swap 영역 관리 최적화 

- 페이지는 수백~수천 개가 생길 수 있다.
- 페이지 크기로 `slot`에 저장하는데 근데 `disk`에 어떤 `slot`을 사용중이고 어떤 `slot`이 비어있는지 확인을 배열로 관리하면 비효율적 
- 따라서 이러한 관리를 1bit로 표현하는 것. ex. 0 ➡ 비어 있음, 1 ➡ 사용 중 

---
### 3.2.  장점 - 공간 절약 (vs 정수형 배열)

> 최소한의 자원으로 큰 용량 처리 가능 

- 디스크의 용량이 클수록 관리해야 할 sector, slot 개수가 매우 늘어난다.
- 근데, 이를 관리하기 위해 4byte인 정수형으로 관리(0, 1 표현 위해)하는 것에 비해 매우 공간을 절약할 수 있다.

> [!INFO] swap 디스크가 8GB 일 경우
> - 총 byte 수 = 8×1,0243 바이트=8,589,934,592 바이트
> - 총 섹터 수 = 총 byte 수 / 512 byte(한섹터) = 16,777,216 개 
> - 이 정도로 많은데 섹터 수가 많은데 배열로 확인하는 것은 비효율적 
> 	- 정수형 배열 : slot당 최소 4byte 必
> 	- 비트맵 : slot 당 1 bit 
> 	- **즉, 정수형 배열에 비해 공간을 최소 32배 절약 가능 ⭐**

---
### 3.3.  빠른 검색 
> Bit 연산은 매우 빠르다.(괜히 비트 마스킹이 알고리즘 성능 좋은게 아님)


---
### 3.4.  캐시 히트 향상 
> Bitmap은 **캐시 지역성이 매우 높다.**

Bitmap은 **메모리 상 연속적인 하나의 거대한 비트열**로 존재한다. 즉, slot의 상태 정보를 매우 조밀하게 모아 놓기 때문에 **캐시 지역성이 높고 cache hit ration가 높아진다.**

만약, 연속적이지 않은 연결 리스트나 해쉬 자료구조를 사용한다면 캐시 히트율이 떨어질 것이다.


---
# 사용 

---
## 1.  사용하는 스켈레톤 코드

---
### 1.1.  create
> bitmap 자료구조를 생성하기 위한 코드

```c
/* Creation and destruction. */
/* Initializes B to be a bitmap of BIT_CNT bits
   and sets all of its bits to false.
   Returns true if success, false if memory allocation
   failed. */
struct bitmap *bitmap_create(size_t bit_cnt) {
  struct bitmap *b = malloc(sizeof *b);
  if (b != NULL) {
    b->bit_cnt = bit_cnt;
    b->bits = malloc(byte_cnt(bit_cnt));
    if (b->bits != NULL || bit_cnt == 0) {
      bitmap_set_all(b, false);
      return b;
    }
    free(b);
  }
  return NULL;
}
```
- `bit_cnt` 개의 비트를 가진 bitmap을 생성.
- 초기에는 모든 비트를 `false`(0)으로 설정.
- swap 영역의 전체 슬롯 수를 기반으로 생성함.

---
### 1.2.  set
```c
  bitmap_set(swap_bitmap, slot_idx, false);
```
```c
/* Atomically sets the bit numbered IDX in B to VALUE. */
void bitmap_set(struct bitmap *b, size_t idx, bool value) {
  ASSERT(b != NULL);
  ASSERT(idx < b->bit_cnt);
  if (value)
    bitmap_mark(b, idx);
  else
    bitmap_reset(b, idx);
}

```
- 특정 인덱스(`idx`)의 비트를 `true`(1) 또는 `false`(0)로 설정.
- swap 슬롯의 할당/해제 시 사용


---
### 1.3.  bitmap_scan_and_flip
```c
  size_t slot_idx = bitmap_scan_and_flip(swap_bitmap, 0, 1, false);  // false 는 빈거
```
```c
/* Finds the first group of CNT consecutive bits in B at or after
   START that are all set to VALUE, flips them all to !VALUE,
   and returns the index of the first bit in the group.
   If there is no such group, returns BITMAP_ERROR.
   If CNT is zero, returns 0.
   Bits are set atomically, but testing bits is not atomic with
   setting them. */
size_t bitmap_scan_and_flip(struct bitmap *b, size_t start, size_t cnt,
                            bool value) {
  size_t idx = bitmap_scan(b, start, cnt, value);
  if (idx != BITMAP_ERROR) bitmap_set_multiple(b, idx, cnt, !value);
  return idx;
}
```
- 지정된 구간에서 연속된 `cnt`개의 비트가 `value` 상태인 구간을 탐색 후,  
    해당 비트들을 반대로(`!value`) 변경.
- swap 슬롯을 **찾고 동시에 예약**할 때 사용.


---
## 2.  swap disk에 사용하기
자세한 내용은 `swapping`에서

링크 : [[Pintos/Pintos VM/Pintos VM - Swapping\|Pintos VM - Swapping]]