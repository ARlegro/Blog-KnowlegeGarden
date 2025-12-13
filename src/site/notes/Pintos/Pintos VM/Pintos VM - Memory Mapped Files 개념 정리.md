---
{"dg-publish":true,"permalink":"/Pintos/Pintos VM/Pintos VM - Memory Mapped Files 개념 정리/","noteIcon":"","created":"2025-12-03T14:52:52.769+09:00","updated":"2025-12-13T09:26:33.304+09:00"}
---




file-backed page의 핵심 주제
### 0.1.  목차
- [[#1.  Mmap이란?|1.  Mmap이란?]]
- [[#2.  Pintos에서 MMAP 시스템 콜|2.  Pintos에서 MMAP 시스템 콜]]
- [[#3.  Thinking 🤔|3.  Thinking 🤔]]
- [[#4.  첫 구현(미완성 작품)|4.  첫 구현(미완성 작품)]]
- [[#5.  mmap 트러블 슈팅|5.  mmap 트러블 슈팅]]

---
## 1.  Mmap이란?

> **파일을 메모리 주소 공간에 직접 매핑**하여 사용하는 방식 

- 파일의 일부를 메모리에 올리고 프로세스의 가상 메모리 영역에 연결한다(Page 단위로 매핑)
>[!tip] VM_FILE타입의 페이지
- **파일**이 `read`/`write`로 접근하는게 아니라 **메모리 접근**으로 가능 
	- **시스템 콜 vs 메모리 접근** : 메모리 접근이 훨씬 빠르다. 또한 메모리 방식에서는 Lazy Load가 가능해서 굿 


---
## 2.  Pintos에서 MMAP 시스템 콜 
Pintos에서는 `mmap()`, `munmap()`시스템 콜을 통해 Memory Mapping을 지원한다.

```C
void *mmap(void *addr, size_t length, int writable, int fd, off_t offset) {
}
```
```C
void munmap(void *addr) 
```


---
## 3.  Thinking 🤔
`mmap()` 생각 
- 결국 이것도 그냥 메모리에 `Lazy_Loading`한다는 거자나?
- `mmap()`만 보면 여태껏 했던 Stack_Growth, ELF 로딩, COPY 등에서 사용했던 로직들 그냥 잘 조합만 하면 될 것 같다.
- 별거 아니라고 생각한다. 일단 이것도 `Lazy_Load` 로직 부분 복사해서 간단히 수정해보면 될 듯?

`unmmap()`은 언제할까❓
- `mmap()`을 만들고 나서 뭘 해제해야할지 생각이 날 것이다. 
- 일단 단서들이 없을 때는 뭐라도 해서 단서들을 만들고 모아가야 할 것 같다. 
- 따라서 이거는 `unmmap()`구현은 `mmap()`을 대충이라도 구현해놓고 시작해봐야 할 것 같다.

---
## 4.  첫 구현(미완성 작품) 

> 첫 구현은 굉장히 단순했다. 단순히 Load관련 로직만 복사해서 일부만 수정했기 때문이다.<br>
> 하지만 "unmmaped와의 연계 + TDD 기반 수정 + 모듈화 Refac"하니까 코드가 굉장히 길어졌다. 그래도 그렇게 어렵지 않았다.

초기 구현 코드
```c
// stick out -> 페이지 단위 작업시 페이지 끝 튀어나오는 부분 0으로 처리 필
void* do_mmap(void* addr, size_t length, int writable, struct file* file,
              off_t offset) {
  ASSERT(addr != 0 || addr != NULL || length > 0 || pg_ofs(addr) == 0);

  struct lazy_load_aux* aux = NULL;
  off_t f_length = file_length(file);
  size_t read_bytes = (f_length > length) ? length : f_length;
  size_t total_zero_bytes = PGSIZE - (read_bytes % PGSIZE);
  // int need_page_cnt = (read_bytes + PGSIZE - 1) / PGSIZE; //최적화 할 때 쓸거
  void* base_addr = pg_round_down(addr);
  void* upage = base_addr;

  struct file* reopened_file = file_reopen(file);
  if (reopened_file == NULL) return NULL;

  while (read_bytes > 0 || total_zero_bytes > 0) {
    size_t page_read_bytes = read_bytes < PGSIZE ? read_bytes : PGSIZE;
    size_t page_zero_bytes = PGSIZE - page_read_bytes;
    upage = pg_round_down(upage);

    /* TODO: Set up aux to pass information to the lazy_load_segment. */
    aux = malloc(sizeof(struct lazy_load_aux));
    if (aux == NULL) return NULL;
    aux->file = reopened_file;
    aux->page_read_bytes = page_read_bytes;
    aux->ofs = offset;
    aux->is_writable = writable;
    aux->page_zero_bytes = page_zero_bytes;
    aux->is_reopened = true;
    if (!vm_alloc_page_with_initializer(VM_FILE, upage, writable,
                                        lazy_load_segment, aux)) {
      goto err_cleanup;
    }

    struct page* page = spt_find_page(&thread_current()->spt, upage);
    if (page == NULL) goto err_cleanup;

    struct mmap_region* region = malloc(sizeof(struct mmap_region));
    if (region == NULL) goto err_cleanup;
    page->mmap_region = region;
    region->file = aux->file;
    region->base_addr = base_addr;
    // region->total_mapped_cnt = need_page_cnt;
    region->addr = upage;
    list_push_back(&thread_current()->mmap_list, &region->elem);

    read_bytes -= page_read_bytes;
    total_zero_bytes -= page_zero_bytes;
    offset += page_read_bytes;
    upage += PGSIZE;
  }
  return addr;

err_cleanup:
  free(aux);
  do_munmap(addr);
  return NULL;
}

static bool has_lazy_load_aux(struct page* page) {
  return VM_TYPE(page->operations->type) == VM_UNINIT &&
         page->uninit.aux != NULL;
}

static void destroy_mmaped(struct mmap_region* region) {
  struct thread* t = thread_current();
  struct page* page = spt_find_page(&t->spt, pg_round_down(region->addr));

  if (page != NULL) {
    if (has_lazy_load_aux) free(page->uninit.aux);
    spt_remove_page(&t->spt, page);  // 얘가 pml4_clear, frame 해제도 해줌
  }
  list_remove(&region->elem);
  free(region);
}

static void remove_related_regions(void* base_addr) {
  struct list* mmap_list = &thread_current()->mmap_list;
  struct list_elem* e = list_begin(mmap_list);
  // 쓰레드가 가진 mmap_list돌면서 관련된 영역에 매핑된 page들 unmap
  for (e = list_begin(mmap_list); e != list_end(mmap_list);) {
    struct list_elem* next = list_next(e);
    struct mmap_region* region = list_entry(e, struct mmap_region, elem);
    // base_addr이 같으면 mmap시 같이 mapping된걸로 간주하고 삭제
    if (region != NULL && region->base_addr == base_addr) {
      destroy_mmaped(region);
    }
    e = next;
  }
}
/* Do the munmap */
void do_munmap(void* addr) {
  if (addr == NULL) return;

  struct thread* t = thread_current();
  struct mmap_region* region = NULL;
  struct page* page = NULL;
  struct file* file = NULL;

  if (list_empty(&t->mmap_list)) return;

  page = spt_find_page(&t->spt, pg_round_down(addr));
  if (page == NULL || page->mmap_region == NULL) return;

  region = page->mmap_region;
  file = region->file;

  remove_related_regions(region->base_addr);
  file_close(file);
}
```

> 이런 코드를 토대로 테스트 코드 돌려가면서 수정해 나갈 것이다.

---
## 5.  mmap 트러블 슈팅 정리 
> 테스트 코드를 기반으로 mmap관련 코드를 완성해 나갔기 때문에 **거의 테스트 코드에서의 트러블**이었다.

링크 : [[Pintos/Pintos VM/Pintos VM - mmap 테스트 트러블\|Pintos VM - mmap 테스트 트러블]]


