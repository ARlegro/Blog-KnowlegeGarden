---
{"dg-publish":true,"permalink":"/Computer_Science/Network/개념/File Descriptor/","noteIcon":"","created":"2025-12-03T14:52:46.209+09:00","updated":"2025-12-13T09:26:25.423+09:00"}
---



### 0.1.  파일 디스크립터(FD, File Descriptor)
#식별자 
![Pasted image 20250924172351.png](/img/user/supporter/image/Pasted%20image%2020250924172351.png)
---
#### 0.1.1.  개념 
> 열린 file/자원을 가리키는 정수 번호 = 식별자 

운영체제에서 프로세스가 어떤 파일(혹은 자원)을 열면, **정수 번호 하나**를 반환하는데, 이 번호가 바로 **파일 디스크립터(File Descriptor, FD)** 
```C
int fd = open("data.txt", O_RDONLY);
```
- 모든 I/O 자원에 적용할 수 있는 번호이다.
- 시스템에서 **파일 객체들을 접근할 때** 사용된다.

>[!tip] UNIX에서 모든 것은 파일이다
>UNIX의 모든 객체들은(정규파일, 소켓, 파이프 등) 모두 "파일"로써 관리됨 

---
#### 0.1.2.  얻게 되는 과정 
`int fd = open("data.txt", O_RDONLY)`
1. 프로세스가 `open()` 시스템 콜을 호출	  
2. Kernel은 파일 테이블에 **해당 파일에 대한 정보를 만든다**
	- Kernel이 오픈 파일(`Struct file`) 객체를 만들고,
	- 거기에 offset, 접근 모드, 파일 연산 테이블 등을 넣는다.
	  
3. 그 파일을 지칭하는 **정수 인덱스를 프로세스에 반환**하여 File Descriptor 테이블에 연결한다  <BR>![Pasted image 20250829133242.png](/img/user/supporter/image/Pasted%20image%2020250829133242.png)
	- 이 인덱스는 Open FIle Table을 가리킴 : 이 안에는 파일 offset, 접근 모드, 메타데이터 등이 있음 
	- 커널의 open file table은 모든 프로세스가 **공유하는 전역 테이블** <br>![Pasted image 20250902193230.png](/img/user/supporter/image/Pasted%20image%2020250902193230.png)<br>![Pasted image 20250902193550.png](/img/user/supporter/image/Pasted%20image%2020250902193550.png)

예를 들어, 프로세스가 특정 입출력 자원과 통신할 때 OS는 File Descriptor라는 식별자 핸들을 준다.

---
#### 0.1.3.  FD의 장점 
1. **추상화된 인터페이스**
    - `read(fd, buf, size)`, `write(fd, buf, size)`만 알면 됨.
    - 개발자는 디스크인지, 소켓인지 신경 쓸 필요 없음.
        
2. **통일성**
    - “모든 것은 파일이다”라는 개념 덕분에, FD는 모든 I/O 자원을 동일한 방식으로 다룰 수 있음.

