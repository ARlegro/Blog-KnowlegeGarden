---
{"dg-publish":true,"permalink":"/Computer_Science/Virtual_Memory/Fork를 사용한 프로세스 생성/","noteIcon":"","created":"2025-08-22T01:24:27.700+09:00","updated":"2025-09-05T02:39:42.585+09:00"}
---



#### 들어가기 전 : 현재 실행 중인 프로세스 확인 
```bash
➜  ~ pstree
systemd─┬─2*[agetty]
        ├─cron
        ├─dbus-daemon
        ├─init-systemd(Ub─┬─SessionLeader───Relay(321)─┬─2*[cpptools-srv───7*[{cpptools-srv}]]
        │                 │                            └─sh───sh───sh───node─┬─node───12*[{node}]
        │                 │                                                  ├─node─┬─cpptools───21*[{cpptools}]
        │                 │                                                  │      ├─language_server─┬─language_server+++
        │                 │                                                  │      │                 └─15*[{language_s+
        │                 │                                                  │      └─13*[{node}]
        │                 │                                                  ├─node─┬─2*[zsh]
        │                 │                                                  │      └─12*[{node}]
        │                 │                                                  └─10*[{node}]
        │                 ├─SessionLeader───Relay(443)───node───6*[{node}]
        │                 ├─SessionLeader───Relay(613)───node───6*[{node}]
        │                 ├─SessionLeader───Relay(1858)───sh───sh───node─┬─sh
        │                 │                                              └─6*[{node}]
        │                 ├─SessionLeader───Relay(87428)───zsh───pstree
        │                 ├─init───{init}
        │                 ├─login───zsh
        │                 └─{init-systemd(Ub}
        ├─redis-server───6*[{redis-server}]
        ├─rsyslogd───3*[{rsyslogd}]
        ├─systemd───(sd-pam)
        ├─systemd-journal
        ├─systemd-logind
        ├─systemd-resolve
        ├─systemd-timesyn───{systemd-timesyn}
        ├─systemd-udevd───30*[(udev-worker)]
        └─unattended-upgr───{unattended-upgr}
```
- 리눅스 계열에서 `pstree`명령어를 치면 볼 수 있음 
- 맨 위 root에는 `systemd`가 있으며, 이는 가장 먼저 시작되고 다른 서비스들을 시작시키는 프로세스이다.

> 대충 이런 구조로 프로세스가 되어 있다는 것만 알기 

---
### fork()를 사용한 프로세스 생성 
> `fork()`를 사용하여 **별도의 복제 프로세스를 생성**할 수 있다.


- 현재 프로세스의 **복사본인 자식 프로세스를 생성** 
- 💢**부모의 완벽한 복사본은 아니다** : 다른 점들이 몇 개 있다. ex. PID, 보류중인 신호, 메모리 잠금 여부 등 
- **부모-자식 간 주소 공간은 쓰기 전까지는 공유**된다 : 쓰기 시에는 Copy-On-Write 메커니즘으로 
  
이는 C, C++에서 라이브러리 함수로도 제공된다.

---
### fork() 호출 과정 

현재 프로세스가 `fork()`를 호출하면 아래와 같은 순서로 동작한다
1. *새 프로세스 생성 + PID 부여*
	- 커널은 새로운 프로세스를 위한 **여러 가지 자료 구조를 생성**하고 
	- **고유 PID를 부여**
	  
2. *가상메모리 사본 생성*
	- 새로운 프로세스를 위해 기존 프로세스의 가상 메모리 공간과 동일한 사본을 생성한다
	  
3. *페이지 테이블 준비*
	- 부모의 페이지 테이블을 복제 
	- But PTE들이 가리키는 물리 Frame은 공유되고 뭐...  읽기전용만들고 쓰기 시 Copy-On-Write해서 사적 공간... 이렇게 만드는데... 이유 굳이 적을 필요 없고 필요할 때 공부 ㄱ 



`fork()`가 호출된 후, 부모와 자식의 실행은 다음과 같이 나뉜다.
1. 부모와 자식 모두 동시에 실행될 수 있다.
2. 부모는 자식이 완료될 때까지 기다릴 수 있다.

---
## execve()
#시스템콜 

> *다른 프로그램을 현재 프로세스의 위치에 새로 실행*하는 시스템 콜 
- 호출하는 프로세스의 **껍데기는 가만히 두고,**
- 안에 들어있는 프로그램을 **통째로 새로운 프로그램으로 갈아끼우는** 


>[!tip] fork() vs execve()
>- fork() : 현재 프로세스 복제 후 자식 프로세스는 새로 생성 
>- execve() : 기존 프로세스를 대체



