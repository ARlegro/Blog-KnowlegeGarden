---
{"dg-publish":true,"permalink":"/Pintos/Pintos User Prog/Pintos U.P - System Call/","noteIcon":"","created":"2025-12-03T16:03:22.652+09:00","updated":"2025-12-03T17:31:37.815+09:00"}
---




### 0.1.  목차

- [[#1.  시작|1.  시작]]
	- [[#1.  시작#1.1.  개념|1.1.  개념]]
	- [[#1.  시작#1.2.  필요한 이유|1.2.  필요한 이유]]
	- [[#1.  시작#1.3.  System Call 내부 흐름|1.3.  System Call 내부 흐름]]
- [[#2.  대표적 System Call|2.  대표적 System Call]]
	- [[#2.  대표적 System Call#2.1.  exit|2.1.  exit]]
	- [[#2.  대표적 System Call#2.2.  wait|2.2.  wait]]
	- [[#2.  대표적 System Call#2.3.  fork|2.3.  fork]]
	- [[#2.  대표적 System Call#2.4.  exec|2.4.  exec]]
	- [[#2.  대표적 System Call#2.5.  open|2.5.  open]]


## 1.  시작

### 1.1.  개념 
> OS가 User Program에게 제공하는 통제된 인터페이스 

- User Program이 H/W, 커널 메모리 접근, 파일 입출력, 프로세스 생성 등과 같은 커널의 핵심 기능을 요청할 때 사용하는 유일한 방법
- System Call은 반드시 커널 모드로 전환하여 실행되어야 한다.


### 1.2.  필요한 이유 
User Progrma은 직접 H/W나 커널 내부에 접근할 수 없다.<br>
대신, OS가 제공하는 System Call 인터페이스를 통해서만 커널 기능을 요청할 수 있다.
(user mode ➡ system call 호출 ➡ 커널 모드로 전환 ➡시스템 콜 핸들로 호출)

>[!QUESTION] 왜 접근을 막아 놨을까❓
>여러 이유가 있겠지만 가장 큰 이유는 "**시스템의 안정성**"을 위한 것이다.
>- 만약 UserProgram이 직접 커널 메모리에 접근할 수 있다면 **버그하나가 커널 자체를 파괴할 수 있다** ➡ 커널이 망가지면 System 충돌 및 비정상 종료 
>- 커널이 유효성을 검증하고 통제된 경로를 통해서만 자원에 접근하도록 함으로써 이를 방지할 수 있다.



### 1.3.  System Call 내부 흐름

시스템 콜의 내부 흐름
1. 유저 프로그램이 **시스템 콜 번호 + 인자**를 레지스터 또는 스택에 세팅
2. `syscall` 명령 실행 → **트랩(trap)** 발생
3. CPU는 사용자 모드에서 **커널 모드로 전환**되고 System Call 엔트리 포인트로 점프
4. 커널이 시스템 콜 번호를 읽고, 인자의 유효성을 검사한 후 **해당 함수(sys_exit, sys_fork, sys_wait, …)** 호출한다.
5. 커널이 작업을 마치고 **반환값을 레지스터에 넣는다**
6. CPU는 사용자 모드로 전환하고, User Program의 다음 명령으로 복귀한다.


---
## 2.  대표적 System Call

`syscall`이라는 **시스템 콜 전용 명령어**를 도입하여 시스템 콜 핸들러를 호출
수 많은 명령어들이 있었지만 핵심 몇 개만 정리하려고 한다.
`exit`, `wait`, `fork`, `exec`, `open`



참고 
- [[Pintos/Pintos User Prog/실제 OS - wait 동기화 매커니즘\|실제 OS - wait 동기화 매커니즘]]
- [[Pintos/Pintos User Prog/Pintos U.P - Process_wait\|Pintos U.P - Process_wait]]
- [[Pintos/Pintos User Prog/Pintos U.P - Fork 개념 정리 + 좀비 프로세스\|Pintos U.P - Fork 개념 정리 + 좀비 프로세스]]

### 2.1.  exit 
> 현재 실행 중인 프로세스의 실행을 영구적으로 중단하고 OS에 제어권을 반환 

- 프로세스가 사용하던 모든 할당 자원(메모리, FD 등)을 커널에 반납한다
- 종료 상태를 부모 프로세스에게 전달한다

### 2.2.  wait
> 자식 프로세스가 종료되기를 기다린다.

- `wait`을 호출한 부모 프로세스는 자식 프로세스가 종료될 때까지 **블록(Block)** 상태로 전환되어 CPU를 사용하지 않는다.
- 좀비 프로세스를 청소해주는 역할을 한다. ([[Pintos/Pintos User Prog/Pintos U.P - Fork 개념 정리 + 좀비 프로세스\|Pintos U.P - Fork 개념 정리 + 좀비 프로세스]])

### 2.3.  fork 
> 현재 실행 중인 프로세스(부모)를 복제하여 새로운 프로세스(자식)를 생성 

- *자원 복제*
	- 자식은 파일 디스크립터와 가상 메모리 공간, 레지스터 상태 등의 **자원들을 복제**
- *반환 값 차이*
	- 부모 : 새로 생성된 자식의 PID를 반환
	- 자식 : 0을 반환 

- *구현 및 트러블 슈팅* : [[Pintos/Pintos User Prog/Pintos U.P - Fork 구현 흐름 정리 및 트러블 슈팅 정리\|Fork 구현 흐름 정리 및 트러블 슈팅 정리]]

>[!tip] 
>자세한거는 fork 정리 및 트러블 슈팅 - [[Pintos/Pintos User Prog/Pintos U.P - Fork 구현 흐름 정리 및 트러블 슈팅 정리\|Fork 구현 흐름 정리 및 트러블 슈팅 정리]] 참고


### 2.4.  exec
> 현재 프로세스의 메모리 공간을 새로운 프로그램의 이미지로 완전히 덮어씌우는 것 

- *프로그램 교체* 
	- PID는 유지된다.
	- 하지만 코드, 데이터, 스택 영역은 지정된 새 실행 파일의 내용으로 모두 초기화된다.
- *일부 자원 유지*
	- open되어있는 FD같은 일부 커널 자원은 그대로 유지된다.

트러블 슈팅 내용 정리
[[Pintos/Pintos User Prog/Pintos U.P - Exec 시스템 콜 구현 및 트러블 슈팅\|Pintos U.P - Exec 시스템 콜 구현 및 트러블 슈팅]]

### 2.5.  open 
> 파일을 열고 파일 디스크립터를 획득한다.



