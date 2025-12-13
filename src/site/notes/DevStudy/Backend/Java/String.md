---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Java/String/","noteIcon":"","created":"2025-12-03T14:52:49.351+09:00","updated":"2025-12-13T10:39:26.208+09:00"}
---


자바 문자를 다루는 클래스는 char, String이 있다.
이 페이지에서는 JAVA의 String에 대해 공부할 것 

## 1.  String 특징
#참조형 

### 1.1.  참조형(Reference Type)
- String은 자바의 클래스이며 **기본형(int, double 등)** 과는 달리 **참조형 타입**이며 **객체를 생성**한다.
- 객체 생성 비교 (vs 기본형)
	```java
	String str1 = new String(”hell”) // 가능 (명시적 객체 생성)
	int 3 = new int(3) // 불가능 ❌ 기본형은 객체로 만들 수 없음.
	```

### 1.2.  문자열 리터럴은 자동으로 객체 처리됨 
```java
String str2 = ”hell”  
 
```
 - 위 코드는 new String("hello")와 유사하게 처리된다.
 - 문자열 리터럴 ➡ **내부적으로 객체 생성**
 - 참고로 String은 보통 아래처럼 문자열 리터럴 방식으로 많이 사용한다.
 - 즉, 자바가 자동으로 인스턴스 형태로 바꿔준다.

---
### 1.3.  불변 객체이다.

String은 불변 객체로 생성되어 있기 때문에 한 번 생성된 문자열 객체의 내부 값이 절대 변경되지 않는다.
String에서는 concat이라는 문자열을 더하는 메서드가 있는데 이 메서드를 써도 이전 String객체의 값은 변하지 않는다. 아래의 예시를 보자: 
```java 
public static void main(String[] args) { 
				String str1 = "hello";
				String str2 = str1.concat("= java");
				System.out.println(str1);
				System.out.println(str2); 

// 결과값 
// hello 
// hello= java
```
- concat을 사용했는데 str1을 보면 문자가 합쳐지지 않았다.
- String은 불변 객체이므로, 변경이 필요한 경우 새로운 객체를 만들어 반환. 위의 예에서는 str2는 str1과 다른 새로운 객체이다.
- concat 메서드가 문자열을 연결하는 역할을 하지만 원본 객체를 수정하지 않고 새로운 String 객체를 생성하여 반환한다.


✅내부 구현 : `String` 클래스의 내부는 다음과 같이 설계되어 있습니다:
```java
final class String {
    private final byte[] value; // 문자열을 저장하는 배열 (final 선언)
```
불변 객체의 핵심은 바로 이 설계 구조에 있음



### 1.4.  동일한 리터럴 공유 by String Pool


>[!tip] 자바는 문자열 풀(String pool) 을 활용해 **동일한 리터럴은 하나의 String인스턴스를 공유**
>![Pasted image 20250621210439.png](/img/user/supporter/image/Pasted%20image%2020250621210439.png)
>- 만약 리터럴 방식으로 String 객체 생성 시 String Poll에 같은 문자열이 있으면 String 인스턴스를 추가 만들지 않고, 그 문자열을 가진 인스턴스의 참조값을 가진다. 
>- 이는, **메모리 효율성을 높이고 성능을 최적화** 할 수 있게 해준다.
>	- Heap 낭비 방지
>	- 객체 생성 비용 감소
>	- 문자열 비교 시 참조 비교로 빠름

> 이로 인해서 같은 문자열 String이 == 연산이 가능한 것 Cuz 같은 참조값을 가지므로 
```java
String a = "hi"
String b = "hi"

a == b // true
```
- "hi"라는 리터럴을 **String Pool에서 검색**
- 이미 존재 ➡**새 객체 생성 X**
- 기존 String 인스턴스의 **참조값만 반환**

>[!QUESTION] String Pool이란❓
>- 문자열 리터럴 전용 캐시 공간
>- 동일한 문자열 리터럴을 중복 생성하지 않고 재사용
>- Heap 영역에 존재


> [!danger] 그래도 문자열 비교시 항상 equals()를 사용해라
> - 개발을 하다보면 뭐가 문자열 풀을 이용하는 변수인지 모름
> - 즉, 문자열은 항상 동등성 비교해라


> [!WARNING] new 로 객체 생성시 무조건 객체를 하나 더 생성
> `new` 키워드는 무조건 새로운 객체를 생성한다(String Pool에 있어도)


## 2.  String 클래스 기능
- 문자열로 처리할 수 있는 여러 메서드 제공
    - 개발자가 편리하게 문자열을 다룰 수 있도록 다양한 기능 제공
    - 엄청 많은데 검색하거나 API 문서를 찾아봐라
- **주요 메서드**
    - `length()` : 문자열의 길이를 반환한다.
    - `charAt(int index)` : 특정 인덱스의 문자를 반환한다.
    - `substring(int beginIndex, int endIndex)` : 문자열의 부분 문자열을 반환한다.
    - `indexOf(String str)` : 특정 문자열이 시작되는 인덱스를 반환한다.
    - `toLowerCase()` , `toUpperCase()` : 문자열을 소문자 또는 대문자로 변환한다.
    - `trim()` : 문자열 양 끝의 공백을 제거한다.
    - `concat(String str)` : 문자열을 더한다


## 3.  왜 불변 객체로 설계했을까??
1. **상수 풀 공유** 
	- String은 자바 내부에서 **문자열 풀**을 통해 동일한 리터럴을 인스턴스로 재사용해 **메모리 최적화**를 한다.
	- **만약, 문자열이 가변**이면 풀의 객체가 변하게 되어 다른 코드에 **사이드 이펙트**를 줄 수 있습니다.
		(같은 문자를 참조하는 변수의 모든 문자가 함께 변경되어 버리는 문제)
	  
2. **스레드 안전**
	- 상태가 바뀌지 않으면 여러 스레드가 동시에 읽어도 문제가 없습니다.

3. **최적화** 
	- equals 비교 시 참조 동일성(`==`) 체크만 하고 빠르게 종료가 가능해 동등성,비교 연산에 최적화

>[!tip] 요약 : `String`을 불변으로 만들면 **메모리 절감, 보안 강화, 스레드 안전, 해시 성능 향상** 등 여러 이점을 한 번에 달성할 수 있다.



## 4.  StringBuilder - 가변 String

> [!warning] 불변 String의 단점
> - 문자를 더하거나 변경할 때마다 새로운 객체를 생성 必 ➡ 많을 경우 메모리, CPU 낭비, 성능저하

> 이러한 String의 단점을 해결하기 위해 StringBuilder

### 4.1.  StringBuilder란?

**✅개념** 
문자열을 변경해도 **새로운 객체를 만들지 않고**, 내부 버퍼를 수정한다.
```java
StringBuilder sb = new StringBuilder();
sb.append("A").append("P").append("P");
System.out.println(sb); // 출력: APP
```

**✅가변 -> 불변** 
메모리 효율을 위해 문자열 **변경 동안에는 가변 객체 사용**하고 **변경 끝나면 불변 객체로 변환**함

**✅생성 법 : StringBuilder로 객체 생성**
- `StringBuilder sb = new StringBuilder();`



### 4.2.  StringBuilder 주요 메서드 
위의 예에서 봤던 `append`는 문자열을 더하는 것인데, 이것 말고도 다양한 메서드들을 지원한다.

| 메서드                            | 설명                   |
| ------------------------------ | -------------------- |
| `append(String s)`             | 문자열 덧붙이기             |
| `insert(int offset, String s)` | 특정 위치에 문자열 삽입        |
| `delete(int start, int end)`   | 특정 구간 삭제             |
| `reverse()`                    | 문자열 뒤집기              |
| `toString()`                   | 최종 결과를 `String`으로 변환 |
사용 예시
```java
public class StringBuilderMain1_1 {
					 public static void main(String[] args) {
					 StringBuilder sb = new StringBuilder();
					 sb.append("A");
					 sb.append("B");
					 sb.append("C");
					 sb.append("D");
					 System.out.println("sb = " + sb);  // ABCD
					 
					 sb.insert(4, "Java");
					 System.out.println("insert = " + sb); //ABCDJava
					 
					 sb.delete(4, 8);
					 System.out.println("delete = " + sb); // ABCD
					 
					 sb.reverse();
					 System.out.println("reverse = " + sb); // DCBA
					 
					 //StringBuilder -> String
					 String string = sb.toString();
					 System.out.println("string = " + string); // DCBA
```

### 4.3.  String vs StringBuilder

| 항목        | String (불변)     | StringBuilder (가변)       |
| --------- | --------------- | ------------------------ |
| 수정 가능 여부  | ❌ 불가능 (새 객체 생성) | ✅ 내부 상태 직접 수정            |
| 성능        | 느림 (객체 매번 생성)   | 빠름 (1개 객체로 수정)           |
| 멀티스레드 안전성 | ✅ (불변이므로)       | ❌ (StringBuffer는 스레드 안전) |
| 사용 시점     | 문자열 변경이 거의 없을 때 | 반복 수정이 많은 경우             |


### 4.4.  String의 최적화 By 자동 StringBuilder 

자바는 문자열 더하는 부분을 자동으로 합쳐준다.
변수도 마찬가지로 최적화 해줌 (이상하면 오류냄) ex.
```java
str1 + st2  
String result = StringBuilder().append(str1).append(str2).toString();
```
- 이렇게 자바가 내부적으로 알아서 **“StringBuilder 변환 - 수정 - String 변환”** 최적화 해줌.


> 따라서, 문자열 합칠 때 쉽게 `+`연산 사용하면 된다.

> [!WARNING] 최적화가 안되는 경우 
> - 문자열을 **반복문 안**에서 문자열을 더하는 경우에는 최적화가 이루어지지 않는다.
> - 루프에서 `StringBuilder 변환 - 수정 - String 변환` 이거를 i번 반복한다는 것은 엄청난 시간 必
> - 따라서, StringBuilder 및 String 변환 은 루프 바깥에 배치시켜야 빠름
```java 
String result = "";
for (int i = 0; i < 1000; i++) {
    result += "A";  // 매 반복마다 새 String 생성 → 매우 비효율적
}
```
### 4.5.  StringBuilder를 직접 사용하는 것이 더 좋은 경우
- **반복문을 통해 문자를 연결할 때** #반복_성능
    - 만약 문자열에서 직접 수정을 한다면 자바가 “StringBuilder 변환 - 수정 - String 변환”이 과정을 i번 반복하기 때문에 느림
    - 따라서 개발자가 따로 StringBuilder를 사용하는게 좋음
		```JAVA 
		StringBuilder sb = new StringBuilder();
		for (int i = 0; i < 1000; i++) {
		    sb.append("A");
		}
		String result = sb.toString();
		```
				
- **조건문을 통해 동적으로 문자열을 조합할 때** : 유연하고 가독성 굿 
- **복잡한 문자열의 특정 부분을 변경해야 할 때** #부분수정
	- `String`은 불변이므로 부분 수정이 불가능 → 새로운 객체 생성
	- 반면 `StringBuilder`는 `insert`, `delete`, `replace` 등을 통해 특정 부분만 효율적으로 수정 가능
- **매우 긴 대용량 문자열을 다룰 때** #전체복사X
    - 긴 문자열에서 매번 새 객체를 생성하면 **메모리 복사 비용이 매우 큼**
    - `StringBuilder`는 내부 버퍼를 확장하며 동작하므로, **메모리 재사용**이 가능하고 훨씬 효율적임


# 메서드 체인닝 - Method Chaining ⭐⭐

### 0.1.  정의
- **연속된 메서드 호출을 체인처럼 연결**하여 코드를 더 간결하게 작성하는 방식
- 메서드가 `this`(자기 자신)를 반환하여 가능
- 함수 반환값에 자기자신을 반환하게 하면 주소값은 그대로 된다.
```java
 public ValueAdder add(int addValue) {
			 value += addValue;
			 return this;

 ValueAdder adder = new ValueAdder();
 int result = adder.add(1).add(2).add(3).getValue(); // 6
```
- adder = x001
- adder.add(1) = x001
- adder.add(1).add(2) = x001

즉, 반환된 참조값을 변수에 담아두지 않아도 되고, 반환된 참조값을 즉시 사용해서 메서드로 호출을 계속 이어갈 수 있다.

### 0.2.  StringBuilder와 메서드 체이닝
StringBuilder는 메서드 체이닝 기법을 제공한다.
`StringBuilder`는 대부분의 메서드가 `this`를 반환 → 체이닝 가능
따라서 StringBuilder 타입은 연결해서 함수 사용 가능
```JAVA 
String result = new StringBuilder()
                    .append("A")
                    .append("B")
                    .insert(2, "Java")
                    .delete(2, 6)
                    .reverse()
                    .toString();
```
