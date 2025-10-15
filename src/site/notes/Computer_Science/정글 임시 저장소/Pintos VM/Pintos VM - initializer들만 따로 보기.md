---
{"dg-publish":true,"permalink":"/Computer_Science/정글 임시 저장소/Pintos VM/Pintos VM - initializer들만 따로 보기/","noteIcon":"","created":"2025-09-30T23:08:00.592+09:00","updated":"2025-10-15T16:23:08.728+09:00"}
---

### 0.1.  목차

- [[#0.1.  Pintos에서 initializer란|0.1.  Pintos에서 initializer란]]
- [[#0.2.  anon_initializer|0.2.  anon_initializer]]
- [[#0.3.  File-Backed 페이지|0.3.  File-Backed 페이지]]
	- [[#0.3.  File-Backed 페이지#0.3.1.  GitBook 내용 및 분석|0.3.1.  GitBook 내용 및 분석]]



### 0.2.  Pintos에서 initializer란

page가 처음으로 메모리에 올라올 때 (frame 확보되어) 무엇을 할지 정해주는 초기화 루틴 

PF ➡ 메모리 Claim ➡ 확보한 frame에 내용 채워 넣고 ➡ ops, 내부 메타데이터 세팅 

즉, `initializer`를 하기 위해서는 그 전에 `vm_do_claim_page`로 커널 VA가 메모리접근하게 frame 확보 必
(frame과 page를 연결 ➡ Initializer)

---
### 0.3.  anon_initializer
#힙 #스택

> 익명 페이지의 초기화 함수 - 메모리 초기화가 핵심 

파일에 연결되지 않고 **단순히 메모리만 사용하는 페이지(ex. 스택, 힙 등)를 위한 초기화 함수**이다.
```c
bool anon_initializer (struct page *page, enum vm_type type, void *kva)
```
- `kva` : `vm_do_claim`에서 할당받은 페이지 `frame`을 가리키는 주소
- anon 타입으로 `kva`를 채우는게 핵심
- 익명 페이지는 `kva`를 전부 0으로 채웠던가? 그리고 project 1,2를 하다보면 `palloc`으로 할당받은 `page`는 `memset()`함수를 통해 값을 채워나갔었다.
- 따라서 익명페이지 초기화 루틴에서는 kva를 0으로 초기화해주는 루틴이 필요할 것 

*추측*
- `anon_page`에 이전에 잠깐 배웠던 slot or sector 번호가 필요할듯?

---
### 0.4.  File-Backed 페이지 
```c
/* Initialize the file backed page */
bool file_backed_initializer (struct page *page, enum vm_type type, void *kva) {
  /* Set up the handler */
  page->operations = &file_ops;

  struct file_page *file_page = &page->file;
}
```

---
#### 0.4.1.  GitBook 내용 및 분석 
*내용*
- `file-backed page`의 내용은 파일에서 가져오므로 `mmap`된 파일을 백업 저장소로 사용해야 합니다. 
- 즉, `file-backed page`를 `evict`하면 해당 페이지가 매핑된 파일에 다시 기록됩니다. 
- `vm/file.c`에서 `file_backed_swap_in`, `file_backed_swap_out`을 구현합니다.
- 여러분의 설계에 따라 `file_backed_init` 및 `file_initializer`를 수정할 수 있습니다.

띵킹 프로세스
- 백업 저장소 생성 必 : mmap된 파일을 백업 저장소로?

아직 딱히 감히 오지 않는다. 듣기로는 `mmap`구현할 때 필요하다는데 아직 mmap은 너무 먼 얘기라 일단은 넘어가면 될 것 같다.


딱히 이 부분은 깊게 파지는 않으려고 한다. 다른 것들을 하다보면 각 page타입별 실행해야 하는 로직들을 느낄 수 있을 것 같다.