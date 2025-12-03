---
{"dg-publish":true,"permalink":"/Pintos/Pintos User Prog/Pintos U.P - Argument Passing/","noteIcon":"","created":"2025-12-03T16:03:22.607+09:00","updated":"2025-12-03T16:08:20.113+09:00"}
---



### 0.1.  Argument Passing (인자 전달)
> 사용자 프로그램에 인자를 올바르게 전달할 수 있도록 하는 것

`./file_name -g -r -p 8080:8082 -e POSTGRES_PASSWORD=postres`
위의 예처럼 프로그램 실행 시에는 단순히 file명만 가지고 실행되는 것은 아니다. 여러 인자들이 들어올 것이고 이를 올바르게 전달해야 한다.

> [!INFO] 
> User Program을 만들 때 스택, 레지스터를 활용해야하는데 이는 **System V AMD64 ABI**를 참고

## 1.  Argument Passing 

### 1.1.  인자 처리 과정
1. **명령어 분리**
2. **문자열 스택에 배치**: 이 단어들을 스택의 상단에 배치
	- argv에 저장된 문자열을 stack에 push (rsp부터)
3. **`NULL` 센티널 넣기 + 정렬하기**
	- 널 포인터 센티넬(`null pointer sentinel`) 을 추가하여 C 표준에 맞게 `argv[argc]`가 널 포인터가 되도록 
	- 또한, 최적의 성능을 위해 첫 번째 푸시 전에 스택 포인터를 8의 배수로 정렬(word-align)하는 것이 좋다.
4. **`argv` 포인터 배열 생성**
	- 각 문자열의 주소를 역순으로 스택에 푸시
    
5. **레지스터 설정**: `%rsi`에 `argv` 배열의 시작 주소(`argv[0]`의 주소)를 설정하고, `%rdi`에는 인자의 개수(`argc`)를 설정
    
6. **가짜 반환 주소 푸시**: 마지막으로, 가짜 "반환 주소"를 푸시하여 스택 프레임 구조를 맞춘다.

### 1.2.  전체 흐름도 

```scss
전체 흐름도
process_create_initd() / syscall_process_execute()
		↓
process_exec() (or syscall_exec())
		↓
tokenize_command_line()  ← 명령어 파싱
		↓
load()  ← ELF 바이너리 로드
		↓
build_stack()  ← 스택 구성 (인자 전달)
		↓
do_iret()  ← 유저 모드 전환
```

### 1.3.  코드 흐름
#### 1.3.1.  tokenize_command_line() 

`int tokenize_command_line(char *command_line, char **argv)`
- 역할: 공백으로 구분된 명령어 문자열을 토큰화
- 처리: strtok_r()을 사용하여 공백 기준으로 분리

#### 1.3.2.  load()

`static bool load(const char *file_name, struct intr_frame *if_)`
- 역할: ELF 실행 파일을 메모리에 로드
- 주요 작업:
	- 페이지 테이블 생성 (`pml4_create()`)
	- 실행 파일 열기 및 ELF 헤더 검증
	- 프로그램 세그먼트 로드 (`load_segment()`)
	- 스택 페이지 할당 (`setup_stack()`)
	- 엔트리 포인트 설정 (`if_->rip = ehdr.e_entry`)

#### 1.3.3.  build_stack()
- 스택 생성
- 스택에 프로그램 인자를 배치하여 `main(int argc, char argv[])`에 전달

상세 단계
```c

  // Step 1: 문자열 데이터 푸시 (역순)
  // argv[argc-1] → argv[0] 순서로 스택에 푸시
  for (int i = argc - 1; i >= 0; i--) {
      void *str_addr = push(interrupt_frame, argv[i], strlen(argv[i]) + 1);
      user_arg_ptrs[i] = (uintptr_t)str_addr;  // 주소 저장
  }

  // Step 2: 스택 정렬 (8바이트 배수)
  align_stack(interrupt_frame);  // RSP를 8의 배수로 정렬

  // Step 3: NULL 센티널 추가
  uintptr_t null_ptr = 0;
  push(interrupt_frame, &null_ptr, sizeof(null_ptr));  // argv[argc] = NULL

  // Step 4: 포인터 배열 생성 (역순)
  // argv[argc-1]의 주소 → argv[0]의 주소 순서로 푸시
  for (int i = argc - 1; i >= 0; i--) {
      push(interrupt_frame, &user_arg_ptrs[i], sizeof(user_arg_ptrs[i]));
  }

  // Step 5: 레지스터 설정
  interrupt_frame->R.rdi = (uint64_t)argc;           // 첫 번째 인자: argc
  interrupt_frame->R.rsi = (uint64_t)interrupt_frame->rsp;  // 두 번째 인자: argv

  // Step 6: 가짜 반환 주소 푸시
  uintptr_t fake_return = 0;
  push(interrupt_frame, &fake_return, sizeof(fake_return));
```

1. 문자열 데이터 푸시 (역순)
	- 스택은 위에서 아래로 자라기 때문에 나중에 push하는 데이터가 더 낮은 주소
	  
2. 스택 정렬
3. `NULL`센티널 추가
4. 포인터 배열 생성
5. 레지스터 설정
6. 가짜 반환 주소 Push
