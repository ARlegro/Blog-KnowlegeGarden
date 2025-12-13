---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Java/Effective Java/Common Method of All Object/Equals를 재정의한다면 HashCode도 재정의해야 한다./","noteIcon":"","created":"2025-12-03T14:52:49.238+09:00","updated":"2025-12-13T10:42:34.325+09:00"}
---



> Equals를 재정의한다면 HashCode도 재정의해야 한다.


## 1.  HashCode를 재정의하지 않으면? (When equals override)


### 1.1.  자바의 규약 위반 

Object 명세에 적힌 규약에 따르면,
"equals()가 두 객체가 같다고 판단했다면, 두 객체의 hashCode는 같은 값을 반환해야 한다"라고 적혀있다.
> 즉, 논리적으로 같은 객체는 같은 해시코드 반환 必

이 규약을 어기면 HashMap, HashSet 등과 같은 해시 기반 컬렉션에서 동작이 비정상적으로 동작합니다.

### 1.2.  규약 어길 시 문제 예시 

hashCode를 재정의하지 않으면 아래의 코드에서 get 사용 시 올바른 value가 나오지 않을 수 있다.
```java
Map<MyClass, String> map = new HashMap<>();

MyClass a = new MyClass("abc");
MyClass b = new MyClass("abc"); // equals()는 같지만, hashCode()는 다름

map.put(a, "hello");

// Expect : "hello"가 출력되길 원하지만 
System.out.println(map.get(b)); // null 반환
```
`hashCode()`가 다르면 `HashMap`은 **서로 다른 key**로 취급 → `get()` 실패

## 2.  좋은 해시 코드란?

"**다른 객체는 다른 해시코드**" 일 확률이 높을수록 성능이 좋다
해시 충돌(collision)을 줄이고, 해시 버킷이 고르게 분포되어야 조회 성능 굿


해쉬코드 구현하는 방법들 쭉 적혀있는데,.... 그냥 IDE쓰자 ㅎ
## 3.  hashCode 필드를 줄이면 생기는 성능 문제 💢
> 해시 시 사용되는 계산을 줄이기 위해 필드를 줄이면 다른 곳에서 성능 문제가 발생한다.
> “해시 품질 저하” = 곧 “성능 저하”

해시 품질이 나빠져 해시 테이블의 성능을 심각하게 떨어 뜨릴 수 있다.
왜냐면 고르게 퍼져서 해시테이블에 배분되는게 아니라, 몇 개의 해시코드만을 가져 특정 해시테이블 인덱스에만 몰려 있을 수 있기 때문이다.
결과적으로 `HashMap`, `HashSet`의 **탐색 성능이 O(1) → O(n)** 로 저하

