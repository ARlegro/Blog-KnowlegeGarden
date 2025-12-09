---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Java/Class Loader 및 로딩 과정/","noteIcon":"","created":"2025-12-07T13:11:50.223+09:00","updated":"2025-12-09T17:19:43.069+09:00"}
---




![Pasted image 20251208144211.png](/img/user/supporter/image/Pasted%20image%2020251208144211.png)

## 1.  Class Loader 개요 및 역할 

### 1.1.  개요 

Class Loader는 JVM의 3가지 구성요소 중 하나이다
- **Class Loader ✅**
- Execution Engine
- Runtime Data Area 



크게는 **바이트 코드(`.class`)를 찾아서 JVM 메모리에 적재**하는 책임을 가진 컴포넌트이다

>[!SUCCESS] 복습 : Java 프로그램 실행과정 
>1. `javac` 이 `.java`를 `.class`의 바이트코드로 컴파일
>2. ✅Class Loader가 `.class`를 찾아서 JVM 메모리에 올림
>3. Execution Engine이 올라온 바이트 코드를 해석하거나 JIT컴파일해서 실제 CPU에서 실행 

> [!WARNING] Class Loader는 소스 코드를 이해하지 않는다.
> 단순히, 바이트 코드들을 찾고 메모리에 올리는 존재 (SRP 잘 지킴)


### 1.2.  역할 

> 이름을 알고 있는 특정 클래스에 대한 정의(byte stream)를 찾아서 읽어오고, JVM 내부 표현(메타데이터)으로 변환하는 것 

1. *이름 기반  Byte Stream 탐색* 
	- 코드에서 `new Util()` 등의 이름을 활용해서 클래스에 대한 정의가 되어 있는 Byte Stream을 찾는다 (JAR 내부 or `.class` or 네트워크)
	- `.class`를 이름만 알면 그거에 대한 byte stream을 가져옴 
	  
2. *Byte Stream -> JVM 내부 구조로 변환*
	- 찾은 Byte Stream이 유효한 포맷인지 **검증**
	- 필드, 메서드, 상수 풀 등의 메타데이터를 정리해서
	- Class 인스턴스로 표현해 JVM 메모리에 적재 (이건 뒤에서 자세히)


가져온다❓ => 네트워크에서 가져올 수도 있다는 뜻 (Java의 확장성 높여주는 역할)
```java
// 예시 
URL[] urls = {
		new URL("http://example.com/libs/util.jar")
};

URLClassLoader loader = new URLClassLoader(urls);
Class<?> clazz = loader.loadClass("com.example.Util");
```
- 인터넷에 있는 JAR 파일을 네트워크에서 byte stream 로딩 



## 2.  Class Loader의 3가지 구성 요소 

`.class`, byte code를 누가 JVM안으로 끌고 들어오는지에 대해 이야기 할 것 


### 2.1.  사전 개념 - 부모 위임 구조 (Parent Delegation)
Class Loader들은 3개가 있는데 이들은 부모-자식 구조를 가진다.

(각 Loader별 자세한 내용은 뒤에서)

```text
[Bootstrap ClassLoader]   ← 최상위(부모의 부모)
        ↑
[Platform ClassLoader]    ← 중간
        ↑
[Application ClassLoader] ← 우리가 쓰는 대부분의 코드, 라이브러리
```

클래스 로딩 요청이 왔을 때 흐름을 보면서 이해하기 
```java
new com.myapp.service.UserService();
```
1. 어떤 클래스가 참조(`new`, `static` 등)되면 Application ClassLoader가 요청을 받는다
2. 먼저 **부모에게 위임**한다
	- 이 클래스를 알면 로딩해달라고
3. Platform ClassLoader가 받는다
4. 또 **자기 부모에게 위임**한다
5. Bootstrap Class Loader가 받았는데 모른다면
6. **아래로 내려와서** Platform Class Loader가 찾는데 모른다면
7. Application Class Loader가 자기의 classpath, JAR를 찾아보고로딩한다.

> 즉, 항상 위에서부터 먼저 찾고 없으면 아래에서 찾는 구조 


이렇게 위임 구조를 잡으면 클래스 충돌이 방지된다(특정 부모, 자식이 있으면 거기서 로드하고 끝나고 위임 없음)

### 2.2.  부트스트랩 Class Loader

> JVM이 올라올 때 **가장 먼저 동작하는 최상위 Class Loader**

- JVM 실행에 필요한 최소 클래스들을 로딩
- 이런 최소 클래스들은 JDK 핵심 라이브러리 클래스들을 의미
	```java
	// 예시 
	java.lang.Object
	java.lang.String
	java.lang.Class
	java.util.*
	java.io.*
	
	rt.jar, tools.jar
	```

*정리*
- JVM이 Native로 들고 잇는 최상위 로더로 **JVM 실행에 필요한 코어 클래스를 가장 먼저 로딩**하는 역할을 한다


### 2.3.  Platform Class Loader

> 코어는 아니지만 **플랫폼 레벨에서 제공되는 표준 라이브러리**를 로딩하는 로더

Platform은 보통 Kernel, H/W영역을 의미한다

예시
- JDK가 제공하는 플랫폼 API(ex. `java.sql`, `java.xml`)

### 2.4.  애플리케이션 Class Loader
> 개발자가 작성한 코드 + 의존하는 라이브러리들을 로딩하는 Loader 

Spring, Hibernate, MyBatis, 프로젝트 도메인 코드 전부 애플리케이션 Class Loader의 관리 대상이다.

JVM을 실행할 때, 아래처럼 하는데 이 때, lib안의 .jar 파일들을 읽어오는게 "애플리케이션 Class Loader"
```bash
java -cp myapp.jar:lib/* com.example.Main
```

Load하는 모듈(일부)
```java
jdk.compiler
jdk.hotspot.agent
jdk.pack
jdk.jshell
```

일반적으로 시스템 Class Loader는 이를 말함 


## 3.  왜 Class Loader 라는 계층을 따로 두었는가?

### 3.1.  클래스 중복 방지 
JVM 내부에서 완전히 **동일한 클래스가 중복되어 로드되는 것을 방지**하고 클래스의 일관성을 유지합니다.
- JVM 클래스 식별 기준 : "클래스명 + 로드 담당한 Class Loader 인스턴스"
- **부모-위임 모델**로 인해 클래스가 한 번 로드되면 동일한 Class Loader에서나는 다시 로드되지 않는다.


가령 Bootstrap에서는 프로그램 실행 전 코어 클래스들을 가장 먼저 로딩하는데, 개발자가 String Class를 만들고 로딩 요청하면 이건 무시된다.

### 3.2.  모듈화 
Class Loader는 JVM 코어 시스템을 건드리지 않는다.
단순히, 개발자가 작성한 코드와 외부 라이브러리(JAR)만을 책임진다.

### 3.3.  유연성 제공 

- Class Loader는 네트워크, 파일 시스템 등 **다양한 위치에서 클래스를 가져올 수 있게 설계**되었다.
- 이로 인해, Java는 **Runtime에 프로그램을 확장할 수** 있다.
- 이로 인해 불필요한 즉시 로딩이 아니라 Lazy Loading으로 메모리를 절약할 수 있다.

### 3.4.  보안 - 클래스 위조 방지
> 부모-위임 모델로 인해 자바의 핵심 라이브러리에 대한 강력한 보안을 보장한다

1. *핵심 클래스 보호* 
	- 부트스트랩 Class Loader가 로딩하는 `Object`, `String`같은 핵심 클래스들은 **가장 신뢰할 수 있는 위치(JDK 내부)에서만 로드되도록 갖게하여 핵심 클래스를 보호**한다
2. *클래스 위조 방지*
	- 이미 로드된 클래스는 다시 로드하지 못하도록 방지되어있다.
	- 따라서, 악의적으로 핵심 API를 변조하려해도 이미 로드된 것이 있기에 **악의적 침투를 원천적으로 방지**한다.


## 4.  Class Loading 3단계

1. 로딩 
2. 링킹
	- 검증, 준비, 해석 
	  
3. 초기화 

### 4.1.  로딩 
*Java의 클래스 로딩* 
- 클래스 로딩 및 링킹 과정이 모두 Runtime에 이루어짐
- 실행 성능이 일부 저하될 수 있으나 "**높은 확장성**"과 "**유연성**"을 제공하는 근간이 된다.
	- Cuz 실행할 프로그램 코드를 네트워크에서 수신 가능
	- Cuz 인터페이스만 맞으면 일단은 구체 클래스를 결정하지 않아도 됨 
- 해석 단계에서는 동적 바인딩을 지원할 목적으로 초기화 후 지연될 수 있음(지연 로딩)

### 4.2.  링킹 - 검증 
> JVM 명세가 정하는 규칙과 제약을 만족하는 지 확인(+ 보안위협 검증도 포함)
> - 파일 형식(`.class`)
> - 메타 데이터
> - 바이트코드
> - Symbol

JVM은 H/W를 손상시키지 않게 엄청 검증함


### 4.3.  링킹 - 준비

클래스들에 대한 메타 데이터를 가지고 있는 **클래스 인스턴스(java.lang.Class)가 힙 영역에서 생성**되고 **클래스 변수 메모리를 0으로 초기화**한다.
- final 선언된 변수는 코드에서 정의한 초기값으로 정의
- 아직, 생성자 호출 전 상태이다⭐(new 연산 전)
	- Just, 인스턴스를 new하기 전 준비 상태임

(java.lang.Class는 클래스를 관리하는 Class)
### 4.4.  링킹 - 해석 

상수 풀의 Symbol 참조를 직접 참조로 대체하는 과정 
- 즉, 상수화 되어있는 Symbol참조를 주소로 대체하는 것 
- 이걸 해내는게 "해석" 과정


### 4.5.  초기화 





## 5.  Class Loading 이후 : Heap 영역에 객체 생성 및 초기화

>[!QUESTION] 
Class Loading을 통해 JVM 메모리에 해당 클래스의 **메타데이터**(`java.lang.Class` 객체)를 올려놓은 뒤에는 무엇이 진행될까❓에 대한 내용이다.



### 5.1.  객체 생성 
Class Loading이 완료되면 JVM은 해당 클래스의 구조(필드, 메서드 등)를 알게된다.
이러한 정보를 바탕으로 "객체 생성"을 함 

>[!QUESTION] 객체 생성이란❓
>로딩이 완료된 클래스를 기반으로 **`new`연산**을 통해 **Heap 영역에 실제 데이터 인스턴스를 만들고 생성자를 호출하여 초기화**하는 과정 

JVM은 객체 저장을 위한 메모리 공간을 확보 후 0으로 초기화 한다.

객체 초기화를 위한 구성설정 실시
- 클래스 이름, 메타정보 확인
- GC 세대 나이 
- 객체에 대한 해시코드
생성자 호출 







