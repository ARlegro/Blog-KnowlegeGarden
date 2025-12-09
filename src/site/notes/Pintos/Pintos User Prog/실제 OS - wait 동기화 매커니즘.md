---
{"dg-publish":true,"permalink":"/Pintos/Pintos User Prog/실제 OS - wait 동기화 매커니즘/","noteIcon":"","created":"2025-12-03T16:03:22.741+09:00","updated":"2025-12-09T17:19:49.183+09:00"}
---




Pintos에서의 wait동기화와 실제 OS가 다른 점을 인지하고 쓴 글 

참고 
- [[Pintos/Pintos User Prog/Pintos U.P - Fork 개념 정리 + 좀비 프로세스\|Pintos U.P - Fork 개념 정리 + 좀비 프로세스]]


## 1.  Wait을 통한 부모 - 자식 동기화 
> 자식 종료 시 SIGCHLD + 커널의 wait queue 블로킹 + task_struct를 통한 exit status 전달

실제 OS에서는 아래와 같은 과정을 통해 동기화가 이뤄진다.
1. 자식 프로세스가 종료하면 **커널이 `SIGCHLD` 신호를 부모에게 보낸다.**
2. `exit status`는 커널의 `task_struct`에 저장된다(자식은 좀비 상태로)
3. `wait()`중인 부모는 `SIGCHLD`를 받거나 자식이 깨우면 `exit status`를 읽고 `task_struct`를 제거하여 동기화를 완료한다.


> `wait()`는 좀비 프로세스 생성을 **방지**하는 핵심적인 방법 


## 2.  부모는 언제 wait를 호출하는가❓

호출하는 경우는 너무 많지만 간단히 몇개만 다루겠다.

1. *자식의 종료 상태가 필요할 때* 
	- 자식의 종료 상태를 포함한 로직이 있다면 필요하다.
2. *자식의 생명주기 관리가 필요할 때* 
	- 좀비 프로세스를 깔끔하게 제거하기 위해 호출하는 경우 
3. *부모가 자식을 반드시 기다려야 할 때*
	- 부모가 자식의 결과를 필요로 하는 경우


## 3.  부모가 wait를 호출하지 않으면❓
위의 동기화는 부모가 `wait`를 한다는 가정하에 처리한 동기화이다.
하지만 부모가 `wait()`을 호출하지 않으면❓좀비 프로세스가 남는거 아닌가❓

> Linux에서는 고아 프로세스 처리 매커니즘을 통해 해결한다.

참고 : [[Pintos/Pintos User Prog/Pintos U.P - Fork 개념 정리 + 좀비 프로세스\|Pintos U.P - Fork 개념 정리 + 좀비 프로세스]] (고아 프로세스)

참고 주소에 자세하게 적혀 있지만, 간단하게 애기해보겠다
- 부모 프로세스가 종료되면 자식은 고아 프로세스가 된다.
- 이 고아 프로세스는 PID1이 맡아 청소해준다.
- 이를 통해 고아가 된 좀비 프로세스도 자동으로 사라지게 된다.


## 4.  실제 OS  vs Pintos 좀비 관점 (간단히)
>[!tip] 실제 OS vs Pintos 의 좀비 상태 알림 
>1. *실제 OS* 
>	- **커널이** 부모 프로세스에게 **자식의 종료 관련 Signal을 보냄** 
>	- **자동 정리도 가능** : 시그널 무시 설정 시 커널이 자식 프로세스 종료 시 자동 정리
>	  
>2. *Pintos* 
>	- **`semaphore`를 활용한 동기화 방식**으로 status 확인 시키도록 

>[!tip] 실제 OS vs Pintos 의 좀비 정리
>3. *실제 OS* 
>	- **자식의 PCB(`task_struct`)를 확인**하고 **종료 상태를 가져온다.** 
>	- 이러한 수확(reaped)과정이 일어나면 PID 반납, PCB(`task_struct`) 정리 등을 한다.
>   
>4. *Pintos* 
>	- **부모-자식 관계를 끊는 역할**. 
>	- PCB(TCB) 관리 없음
>	- 💢**PID 반납도 없고 재사용도 안함** 

```C
// 자식 프로세스의 exit()
void thread_exit (void) {

  ASSERT (!intr_context ());
	...
  do_schedule (THREAD_DYING);  // DYING 상태로 만들고 스케쥴러에게 위임 
```

*부모 프로세스에게 알림*
- *Pintos* : semaphore를 활용한 동기화 방식으로 status 확인 시키도록 
- *실제 OS* : 커널이 부모 프로세스에게 자식의 종료 관련 Signal을 보냄 (시그널 무시 설정 시 커널이 자식 프로세스 종료 시 자동 정리)

*정보 수거 및 완전 소멸(부모)*
- **부모는 자식의 종료 상태를 확인하고 PID 반납, PCB 해제 등 남은 기록들을 정리**한다. (실제 OS와 Pintos는 약간 다름)

