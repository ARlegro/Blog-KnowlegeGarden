---
{"dg-publish":true,"permalink":"/Computer_Science/Network/C-Network/Tiny 웹 서버 구현/","noteIcon":"","created":"2025-08-30T15:54:35.065+09:00","updated":"2025-09-05T02:25:16.292+09:00"}
---


참고
- [[Computer_Science/Network/개념/Socket 개념과 간단한 에코 서버\|Socket 개념과 간단한 에코 서버]]
- [[Computer_Science/Network/개념/OSI 7 Layer\|OSI 7 Layer]]
- [[Computer_Science/Network/개념/TCP.IP 4계층 모델\|TCP.IP 4계층 모델]]
- 깃 허브 : https://github.com/ARlegro/web-server-impl

---
### 주요 함수 정리 

---
#### doit(int fd)
> 역할 : 하나의 HTTP 트랜잭션을 처리 

과정 
1. *요청라인을 읽는다.*
2. *GET 메서드인지 확인*
	- 💚GET 메서드라면 ➡ 3번 진행
	- 😞GET 메서드가 아니라면 : GET 메서드를 제외한 다른 메서드 요청 시 오류 메시지를 보내고 메인 루틴으로 돌아간다(`doit()` 종료)
	  
3. *uri를 파싱해서 컨텐츠의 종류 파악(동적 or 정적)*
	- `parse_uri(uri, filename, cgiargs)`라는 URI 분석 헬퍼 메서드를 사용 
		- `?x=1&y=3`이런게 들어가면 동적인 것 
		- return : 정적(1), 동적(0) 
		- *추가 효과* : filename, cgiargs를 uri, param을 반영해서 채워넣어줌
	  
4. *파일의 상태 정보(크기, 종류, 권한 등)을 사전에 정의한 구조체에 채워주기*
	- 예시 : `stat(filename, &sbuf)`
		- `/* Get file attributes for FILE and put them in BUF.  */`
		- stat : 리눅스 표준 시스템 콜 by `<sys/stat.h>`
		- 실패 시 음수 반환 : 파일이 없거나 접근 불가인 경우
	- 이유 : 나중에 컨텐츠 처리 시 必
	  
5. *동적유무에 따라 플래그 나눠서 컨텐츠 처리 *
	1. *정적 컨텐츠인 경우*
		- 일반 파일이고 읽기 권한이 있는지 확인
		- 만약 조건이 맞다면 클라이언트에 콘텐츠 제공
		  
	2. *동적 컨텐츠인 경우*
		- 파일이 실행 가능한지 확인
		- 만약 조건이 맞다면 클라이언트에 콘텐츠 제공

---
#### clienterror(...)
```c
void clienterror(int fd, char *cause, char *errnum, char *shortmsg, char *longmsg);
```
인자
- fd : 파일 디스크립션
- cause : 원인 
- errnum : 상태 코드 

*과정* 
1. HTTP response body를 만든다.
2. HTTP respsonse를 출력한다.

---
#### read_requesthdrs()

> [!WARNING] 참고 : Tiny 서버는 요청 헤더의 어떤 정보도 사용 안함 
> 그래서 이 함수는 읽기만 하고 무시

```c
void read_requesthdrs(rio_t *rp)
// rp : 클라이언트 소켓과 연결된 버퍼 상태 
{ 
		char buf[MAXLINE]; 
		// 첫 줄 읽기 : GET /index.html HTTP/1.1
		Rio_readlineb(rp, buf, MAXLINE); 

	  /**
		 * 나머지 줄 읽기 
		 * Host: localhost:8000
		 * User-Agent: Mozilla/5.0
		 * Accept: text/html
		 * ...
		 */
		while(strcmp(buf, "\r\n")) { 
			 Rio_readlineb(rp, buf, MAXLINE); 
			 printf("%s", buf); 
	  }
 
	  return; 
} 
```
`strcmp(buf, "\r\n")
- 현재 읽은 줄이 빈 줄(`"\r\n"`)인지 확인 
- Rio_readlineb보다 먼저 출현하므로 빈 줄을 한번 읽긴한다. 헤더 마지막에 빈줄을 확인

---
#### parse_uri(uri, filename, cgiargs)
 > *메인 역할*: uri를 파싱해서 **컨텐츠의 종류 파악**(동적 or 정적)
 > *추가*: filename, cgiargs를 **uri, param을 반영해서 채워 넣어줌**

>[!tip] 정적, 동적 판단 근거 
>uri에 "cgi-bin"문자열이 **포함되어 있지 않으면 정적 컨텐츠**로 간주
>- `?x=1&y=3`이런게 들어가면 cgi문자열이 있어서 동적인 것 


```C
// return : 1 - static, 0 - dynamic
int parse_uri(char *uri, char *filename, char *cgiargs){

  char *ptr;

  if (!strstr(uri, "cgi-bin/")){  /* 정적 컨텐츠인 경우*/
    strcpy(cgiargs, ""); // 1. cgi인자 초기화
    strcpy(filename, ".");
    strcat(filename, uri);

    // 만약 경로가 '/'로 끝나면 'home.html'경로 추가
    if (uri[strlen(uri)-1] == '/'){
      strcat(filename, "home.html"); // default page
    }
    return 1;
  }

  else { /* 동적 컨텐츠인 경우 */
    // GET /cgi-bin/sum?k1=v1&k2=v2   <<< d이 경우 ?이 처음 나오는 위치의 인덱스
    ptr = index(uri, "?");
    if (ptr) {
      // ? 뒤에 나오는 문자를 cgiargs에 쓰기
      strcpy(cgiargs, ptr+1);
      *ptr = '\0';
    }
    else {
      // cgiargs 초기화 - 어차피 없으니
      strcpy(cgiargs, "");
    }
    strcpy(filename, ".");
    strcat(filename, uri);
    return 0;
  }
```

🛑*정적 컨텐츠인 경우 *
1. *cgiargs 문자열을 비워준다.* - `strcpy(cgiargs, "")`
	- 동적 컨텐츠가 아니기 때문에 쿼리 인자가 필요가 없음 
	  
2. *filename을 작성해줌 - `strcpy(filename, "."), strcat(filename, uri)`*
	- uri가 `/cgi-bin/sum` 이면 filename은 `.`을 붙여서 `./cgi-bin/sum` 이렇게 됨 
	  
3. *Default 문서 처리*
	- uri 끝에 `/`가 붙으면 `home.html` 경로 추가 

	  
✈*동적 컨텐츠인 경우*
	1. *쿼리 문자열 위치 탐색 - `index`*
	2. *cgiargs 쓰기*
	3. *filename 작성해주기*

return : 정적(1), 동적(0) 

---
#### serve_static
> 정적 컨텐츠 다루는 함수 
> - 동적 컨텐츠와 달리 현재 프로세스가 처리 

```c 
void serve_static(int fd, char *filename, int filesize){

  char filetype[MAXLINE], buf[MAXBUF];
  char *source_pointer;
  
  // 1 
  get_filetype(filename, filetype); 
 
  sprintf(buf, "HTTP/1.0 200 OK\r\n");
  sprintf(buf, "%sServer: Tiny Web Server\r\n", buf);
  sprintf(buf, "%sConnection: close\r\n", buf);
  sprintf(buf, "%sContent-length: %d\r\n", buf, filesize);
  sprintf(buf, "%sContent-type: %s\r\n\r\n", buf, filetype);
  // 2. 클라이언트에 HTTP response header 데이터 전송
  Rio_writen(fd, buf, strlen(buf));

  printf("Response header:\n");
  printf("%s", buf);

	// 3. file Open -> 식별자 가져오기 
  int source_descriptor = Open(filename, O_RDONLY, 0);

	// 4. file을 가상 주소 공간에 매핑 
	// 처음 0 = 매핑할 메모리의 시작 주소를 힌트로 지정
  source_pointer = Mmap(0, filesize, PROT_READ, MAP_PRIVATE, source_descriptor, 0);

	// 5. file 식별자 닫기 
  Close(source_descriptor);

	// 6. 소켓에 바로 쓰기 (가상 메모리 매핑 상태)
  Rio_writen(fd, source_pointer, filesize);

	// 7. 가상 메모리 매핑한 것 해제 : 리소스 반환
  Munmap(source_pointer, filesize);
```

1. *파일 타입을 얻는다 - `get_filetype(filename, filetype)`*
2. *클라이언트에게 응답 헤더를 보낸다*
	- sprint와 buf를 활용해서 누적하고 마지막에 Rio_writen 방식 ㄱ 
	- 마지막에는 빈줄 만들어야 해서 `\r\n\r\n`
	  
3. *file을 오픈하여 식별자를 얻어온다* - `Open(const char *filename, int flags, mode_t mode)`
	- 식별자를 곧바로 사용하지는 않을거고 4번을 위한 초석 
	  
4. *file을 프로세스 가상주소공간에 바로 매핑 - `Mmap`*
	- file을 메모리처럼 읽을 수 있게 함 
	  
5. *3번에서 얻은 식별자를 닫는다*
	- 4번에서 가상메모리로 mapping했으니 파일 데이터를 읽는 데 fd가 필요가 없어진다
	- **Close하는 이유** 
		- 식별자는 단순히 숫자가 아니라, **커널에 있는 파일 객체를 가리키는 핸들**이다.
		- 처음에 커널은 파일을 Open하고 해당 파일 상태를 담은 구조체(open file entry)를 하나 만든 상태인 것 
		- 만약, 이 **Close 작업을 하지 않으면** 해당 구조체(엔트리)가 커널 테이블에 해당 구조체가 남아 있어서 **리소스 누수가 생김** 
	  
6. *가상메모리 매핑을 바탕으로 소켓에 file내용을 흘려보낸다*
7. *4번에서 매핑한 것을 해제하고 리소스를 반환한다*


>[!tip] 굳이 가상 메모리 매핑하는 이유 
>1. *❌메모리 매핑을 안 한다면?*
>	- `while (n = read(~~~) > 0` 이런 로직을 짜면서 여러 번의 `read()`/`write()`를 반복할 것이다.
>	- 이로 인해 수많은 System Call 발생 😞
>	  
>2. *✅메모리 매핑을 한다면?*
>	- 프로세스 입장에서는 **file이 그냥 연속된 긴 배열로 보이기 때문**에 `read()`없이 바로 `write()`로 파일 내용을 소켓에 흘려보낼 수 있다.
>	- 페이지 캐시를 이용해서 System Call을 여러 번 할 필요가 없다

---
#### serve_dynamic()
> 동적 컨텐츠를 클라이언트에 제공하는 함수 
> - 메인 로직은 서버 말고 **다른 프로그램(CGI)이 처림** 

![Pasted image 20250831115625.png](/img/user/supporter/image/Pasted%20image%2020250831115625.png)
```c
void serve_dynamic(int fd, char *filename, char *cgiargs){

  char buf[MAXLINE], *emptylist[] = {NULL};
  sprintf(buf, "HTTP/1.0 200 OK\r\n");
  sprintf(buf, "%sServer: Tiny Web Server\r\n", buf);
  // 1 
  Rio_writen(fd, buf, strlen(buf));

	// 2
  if (fork() == 0){
    /** 3
     * setenv 이유
     * - Excecve의 environ은 방금 설정한 환경을 포함하는데,
     * - CGI 내부에서 getenv("QUERY_STRING")으로 값을 읽을 수 있다.
     * - QUERY_STRING : ?뒤의 쿼리 문자열 value를 담는 키
     */
    setenv("QUERY_STRING", cgiargs, 1);

    /** 4
     * Dup2(oldfd, newfd)
     * - 정의 : newfd를 oldfd(클라 소켓)의 복사본으로 만든다.
     * - 효과 : CGI 프로그램이 표준출력으로 쓰는 모든 데이터가 곧바로 oldfd로 전송됨
     * - STDIN_FILENO = 1  by unistd.h
     */
    Dup2(fd, STDOUT_FILENO);
    /** 5
     * CGI 실행 및 프로세스 이미지 교체
     * - path : 실행할 스크립트의 경로(CGI 프로그램의 경로)
     * - environ : 현재 프로세스의 환경 포인터
     */
    Execve(filename, emptylist, environ);
  }
	/* 6 이걸 호출한 프로세스(부모)의 자식 프로세스가 종료될 때까지 기다림 */
  Wait(NULL);
```


1. *HTTP response의 앞 부분을 짧게 `writen()`한다*
	- 자세한거는 복제 프로세스(CGI)에서 처리
	  
2. *자식 프로세스를 복제한다 - `Fork()`*
	- 현재 실행중인 프로세스를 복제하는 ㅅ시스템 콜 
	- Return값은 자식 프로세스의 PID (초기에는 0)
	  
3. *복제한 자식 프로세스에 환경변수를 주입한다 - `setenv()`*
4. *CGI 프로그램의 표준 출력을 클라이언트 소켓의 복사본으로 만든다* - `Dup2()` 
	- CGI의 표준출력을 소켓으로 연결하는 것 
	- CGI가 표준 출력으로 쓰는 모든 Byte가 소켓으로 전송됨
	  
5. *CGI 실행 및 프로세스 이미지 교체 - `execve()*
	- 프로세스를 **다른 프로그램으로 교체**한다.
	- `execve`
		- **새로운 프로그램을 실행**하고 다른 프로그램으로 **갈아끼우는** System call 
		- 현재 실행 중인 프로세스의 code, data, stack, heap을 **새로운 프로그램의 code와 data로 완전 대체**한다.
		- 💢But, 기존 프로세스의 일부(PID, 환경변수)는 유지된다.
	- *함수 분석*
		```c
		execve(const char *path, char *const argv[], char *const envp[])
		```
		- *path* : **실행할** 바이너리 or **스크립트의 경로**  ex. `./cgi-bin/adder, /usr/bin/python`
		- *argv* : 인자 리스트. argv[0]은 관례상 프로그램 이름
		- *envp* : 환경변수 리스트. 그냥 전역변수이자 **현재 프로세스의 환경 포인터**인 `environ` 넘기면 환경이 새 프로그램에 전달됨


	- `environ` : **프로세스 전체의 환경 변수 목록**을 가리키는 **전역 변수**.


>[!tip] 표준 패턴 : fork ➡ 자식 환경 세팅 ➡ execve

>[!question] 동적 컨텐츠 제공 시 fork()로 자식 프로세스를 만드는 이유 
>- tiny 서버는 CGI로 동적 컨텐츠를 다루는데, **CGI는 서버 코드와 완전히 다른 실행 context를 가진 프로그램**이다.
>- 만약, 서버 프로세스에서 CGI를 실행하면 **잘못된 공유(소켓 fd, buffer, 주소 공간 등)가 일어나 충돌이 일어날 수 있다.** 💢
>- 이런 일을 방지하고자, fork()로 서버 프로세스를 그대로 복제하고, 자식 프로세스만 CGI 프로그램으로 덮어씌우는 것 
>- 이로 인해, 요청 관리하는 서버 프로세스와 실행하는 CGI 프로세스가 **분리되어 안정성이 높아진다**
>- `fork()` Return : -1 for errors, 0 to the new process



