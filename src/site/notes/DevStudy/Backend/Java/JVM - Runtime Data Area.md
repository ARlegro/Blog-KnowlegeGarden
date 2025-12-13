---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Java/JVM - Runtime Data Area/","noteIcon":"","created":"2025-12-09T17:19:43.303+09:00","updated":"2025-12-13T09:26:27.045+09:00"}
---



![Pasted image 20251208144226.png](/img/user/supporter/image/Pasted%20image%2020251208144226.png)
출처 : https://velog.io/@wkdwoo/JVM-Runtime-Data-Area


## 1.  개요 및 역할 

> JVM이 프로그램을 수행하기 위해 OS로부터 할당받는 메모리 영역 


### 1.1.  구조적 특징 
![Pasted image 20251208144330.png](/img/user/supporter/image/Pasted%20image%2020251208144330.png)
- **일부 영역은 스레드마다 독립적으로 생성**된다
	- PC Register
	- JVM Stack
	- Native Method Stack
	  
- **일부 영역은 모든 스레드가 공유**한다
	- Heap
	- Method Area
	- Runtime Contant Pol


### 1.2.  대부분의 WAS 성능 원인 = Heap 관리 문제 

WAS성능의 문제는 Runtime Data Aread의 Heap영역이 대부분이다.
- 메모리 누수
- 객체 과할당
- Old 영역 증가
- Full GC 증가

> 따라서, 이 영역에 대한 개념을 아는 것이 중요




## 2.  스레드별 생성되는 영역

JVM Stack, PC Register, Native Method Area는 쓰레드마다 생긴다.


### 2.1.  PC Register
> 현재 실행중인 스레드가 **다음에 실행해야 할 명령어의 주소를 저장**하는 역할 


JVM은 **스택 기반이라 다음 명령 위치를 별도로 저장해야**하는데 이곳이 PC Register
- 다음 명령어 주소를 JVM에 알려주어 CPU가 순서대로 명령을 실행할 수 있도록 함
- 모든 스레드가 독립적인 PC Register를 가진다.

### 2.2.  JVM Stack 
> Java 메서드가 호출될 때마다 생성되는 Stack Frame을 저장하는 공간


- *저장되는 것*
	- 지역변수 테이블, 피연산자 스택, 메서드 반환값 등 
- 스택의 크기는 정해져있고 이를 **초과한다면 StackOverFlow 에러**가 발생

### 2.3.  Native Method Stack 

> JVM이 아닌 **native 언어 (C/C++)로 작성된 메서드가 호출될 때** 사용하는 스택 메모리

JNI(Java Native Interface) 함수 호출 시 사용된다.


JVM 스택과의 차이

| 항목       | JVM Stack          | Native Method Stack   |
| -------- | ------------------ | --------------------- |
| 실행 코드    | Java 바이트코드         | C/C++ 등 Native 코드     |
| Frame 구조 | JVM이 고정 정의         | 플랫폼에 따라 다름            |
| 오류       | StackOverflowError | StackOverflow + OS 의존 |

## 3.  모든 스레드가 공유하는 영역

### 3.1.  Method Area

> Class Loader가 로드한 **클래스의 메타데이터가 저장**되는 영역


저장되는 내용 예시
- 클래스 이름, 부모 정보
- 인터페이스 목록
- 필드/메서드 메타데이터
- static 변수
- final static 상수
- *Runtime Constant Pool*
- **JIT 컴파일러**가 번역한 기계어 코드를 **캐싱하기 위한 메모리 공간으로도 사용**된다.

### 3.2.  Runtime constant pool ⭐

> 클래스 파일 내부의 constant pool 정보를 JVM 메모리에 재구성한 테이블

저장되는 항목
- 숫자/문자 리터럴
- 메서드/필드 Symbol 정보
- 런타임에 생성되는 문자열 상수


- 이러한 정보들은 Class Loader가 클래스를 **Load할 때 이 pool에 저장**한다.
- **동적으로 운영**되며 Runtime에 새로운 상수가 추가될 수 있다.
- 하나의 클래스만 해도 엄~청 많은 상수들이 여기에 저장됨 


### 3.3.  힙 영역 ⭐

> JVM에서 GC가 관리하는 객체 저장소 

저장되는 것
- new 로 생성되는 모든 객체 (클래스 인스턴스)
- 배열
- 람다 객체
- JIT 최적화로 Promotion된 코드 일부 



*✅특징* 
1. *GC가 관리한다*
	- Heap은 JVM의 GC가 관리하는 메모리 영역이다.
	- **더 이상 참조되지 않는 객체는 GC에 의해 회수**된다.
2. *설계 및 관리*
	- 효율적인 GC를 위해 객체를 **생성 시점에 따라 구분하여 관리**하는 **"세대별 컬렉션 이론"** 에 따라 설계되었다.(자세한건 JVM GC에서)
3. *에러* 
	- 메모리가 부족할 경우 `OutOfMemoryError`가 발생

## 4.  JVM GC 

[[DevStudy/Backend/Java/JVM - GC\|JVM - GC]]
