---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Java/Effective Java/Common Method of All Object/Override equals in accordance with its general contract/","noteIcon":"","created":"2025-12-03T14:52:49.209+09:00","updated":"2025-12-13T09:26:26.934+09:00"}
---



> Equals는 일반 규약을 지켜 **재정의**하라 

> [!WARNING]  equals 규약을 어기면 그 객체를 사용하는 다른 객체들이 어떻게 반응할지를 알 수 없다. ex. List, Set에서의 contains 등 


>[!tip] 필요한 경우가 아니라면 Object의 equal로도 충분한 비교가 가능하다. 따라서 필요한/필요하지 않은 경우를 알고 equals를 Override해라

### 0.1.  ❌재정의(Override)하지 않는 것이 좋은 경우 

1. **각 인스턴스가 본질적으로 고유할 경우**
	- 같은 인스턴스가 둘 이상 만들어지지 않음을 보장할 때 ex. Enum
	  
2. 인스턴스가 **논리적 동치성(logical equality)을 검사할 일이 없는** 경우
3. **상위 클래스에서 Override한 equals가** 하위 클래스에서도 **잘 적용**되는 경우 
4. 클래스 자체가 private, package-private이라 **equals메서드를 호출할 일이 없는** 경우


### 0.2.  ✅재정의(Override)하는 것이 좋은 경우 

>[!tip] 객체 식별(identity) 목적이 아니라 **논리적 동치성(equality)을 확인해야** 할 때

대표적인 예로는 값 클래스이다(Integer, String)
- 이런 값 클래스들은 equals가 재정의 되어 있다.
- **효과** : Map의 Key Set의 원소로 사용할 수 있게 됨 
	- 재정의 되어 있지 않으면 key값이 논리적으로는 같은데 다른 key로 인식

---
## 1.  Object 명세 규약 

>equals 메서드는 동치 관계(Equivalence relation)를 구현하며 아래를 만족
>(참고 : 아래는 모든 참조 값이 null 이 아니라는 가정이다)

### 1.1.  반사성 
>모든 참조 값 x에 대해 x.equals(x)는 true이다.

객체는 자기 자신과 같아야 한다는 뜻 
만족 못 시키는게 이상한 듯?

---
### 1.2.  2.대칭성 
> 모든 참조 값 x, y에 대해, x.equals(y)가 true이면 y.equals(x)도 true이다.

두 객체는 서로에 대한 동치 여부에 똑같이 답해야 한다.
대칭성 위배 예시 : String 파라미터 값을 대소문자 무시하기 위해 equals 구현하는 경우
```java 
public final class CaseInsensitiveString {
		prinvate final String s;
}

❌잘못된 예시
@Override  
public boolean equals(Object obj) {  
    // 이건 그냥 이 클래스일 경우   
if (obj instanceof CaseInsensitiveString) {  
        return s.equalsIgnoreCase(((CaseInsensitiveString)obj).s );  
    }  
      
    // ❌ String일 경우 equals 정의한거 <<< 이게 문제   
if (obj instanceof String) {  
        return  s.equalsIgnoreCase((String) obj);  
    }  
    return false;  
}
```
- CaseInsensitiveString보유한 s와 equals 비교 대상인 obj가 string일 때 대소문자를 구분하지 않으려고 재정의 했다.
- But, 이렇게 하면 `obj.equals(CaseInsensitiveString)`는 false가 되고 `CaseInsensitiveString.equals(obj)`는 true가 되는 이런 비대칭적 결과가 나올 수 있다.
- 위의 경우는 String까지 equals를 연동하겠다는 것은 허황된 꿈이다. 그냥 인스턴스 확인 && equalsIngnoreCase 로 하는 것이 낫다.

> [!WARNING] 이는 equals의 규약인 대칭성을 어긴 것 
> - equals 규약을 어기면 그 객체를 사용하는 다른 객체들이 어떻게 반응할지를 알 수 없다. ex. List, Set에서의 contains 등 

---
### 1.3.  추이성
>모든 참조 값 x, y, z에 대해, x.equals(y)가 true이고 y.equals(z)도 true이면 x.equals(z)도 true이다.


이 규약을 깨뜨리면 컬렉션(특히 `Set`, `Map`), 캐시, ORM 식별자 비교, 테스트 단언 등 “동등성”을 전제한 모든 코드가 깨지거나 예측 불가능하게 동작한다.

#### 1.3.1.  추이성 위반 예시 

**1‍⃣ 상속으로 상태를 확장한 값 객체** 
```java
// 1. 기본 좌표
public class Point {
    private final int x, y;
    @Override public boolean equals(Object o) {
        if (!(o instanceof Point p)) return false;
        return x == p.x && y == p.y;
    }
    @Override public int hashCode() { return Objects.hash(x, y); }
}

// 2. 색상을 추가한 좌표 
public class ColorPoint extends Point {
    private final Color color;
    @Override public boolean equals(Object o) {
        if (!(o instanceof ColorPoint cp)) return false;
        // Point 비교 + 색상 비교
        return super.equals(cp) && color.equals(cp.color);
    }
}
```
```java 
Point p   = new Point(1, 2);
ColorPoint cp1 = new ColorPoint(1, 2, Color.RED);
ColorPoint cp2 = new ColorPoint(1, 2, Color.BLUE);

boolean r1 = p.equals(cp1);   // true (Point 입장에선 색 정보 무시)
boolean r2 = cp1.equals(cp2); // false (색 다름)
boolean r3 = p.equals(cp2);   // true
// 추이성 깨짐: r1 && r2 == true 여도 r3 == false 여야 하는데 true가 나온다
```

발생할 만한 문제 
- ORM 영속성 컨텍스트에서 서로 다른 인스턴스를 같은 ID로 인식 
- Map에서 서로 다른 Key가 동일하다고 판단되어 충돌 

#### 1.3.2.  ✅해결 

> ❌“**값을 상속으로 확장**”하는 설계는 **동등성 규약엔 적합하지 않다.**
> 즉, 추가 상태는 상위 타입이 이해하지 못 하므로 상/하위 타입 모두 만족하는 일관된 equals를 사용할 수 없다는 것 

ColorPoint가 부모로 여겼던 
```java
```java
// 1. 기본 좌표
public class Point {
    private final int x;  
		private final int y;
}

// 2. 상속 없이 필드로 
public class ColorPoint {
		private final Point point 
    private final Color color;
    
		@Override  
		public boolean equals(Object o) {  
		    if (o == null || getClass() != o.getClass()) return false;  
		    ColorPoint that = (ColorPoint) o;  
		    return Objects.equals(point, that.point) && Objects.equals(color, that.color);  
		}
}

```

#### 1.3.3.  상속 값의 equals 실패 다른 예시 

```java
// 1. 값 객체
public class Money {
    private final BigDecimal amount; private final Currency currency;
    @Override public boolean equals(Object o) {
        if (!(o instanceof Money m)) return false;
        return amount.equals(m.amount) && currency.equals(m.currency);

// 2. 할인 금액 
public class DiscountedMoney extends Money {
    private final String promotionCode;   // 추가 상태
    @Override public boolean equals(Object o) {
        if (!(o instanceof DiscountedMoney dm)) return false;
        // 금액·통화 비교 + 프로모션 코드 비교 
        return super.equals(dm)        
               && promotionCode.equals(dm.promotionCode); 
```

```java
Money            price = new Money(10000, KRW);
DiscountedMoney  sale  = new DiscountedMoney(10000, KRW, "SUMMER");

Set<Money> cache = new HashSet<>();
cache.add(price);          // size == 1
cache.add(sale);           // size == 2 (expected)

// 다른 곳
boolean eligible = cache.contains(price); // true
boolean extra    = cache.contains(sale);  // false ❌
```
실제로 cache.add(sale)은 추가되지 않았다. 그렇기에 cache.contains(sale) 또한 false가 나온다.
이유 : 아래의 표

| 시점                                          | equals 호출 방향         | 결과                                 |
| ------------------------------------------- | -------------------- | ---------------------------------- |
| **put 단계**  <br>`cache.add(sale)`           | `price.equals(sale)` | `true` → _중복으로 간주 → sale은 저장되지 않음_ |
| **contains 단계**  <br>`cache.contains(sale)` | `sale.equals(price)` | `false` → _존재하지 않는다고 판단_           |
그러면 어쨋든 Money타입으로 equals 비교이니까 contains는 true여야 하는 것이 아니냐❓ 라는 의문을 가질 수 있는데, 위의 표처럼 put과 contains확인 단계에서 equals의 호출 방향이 달라지는데, 이 때 크게 문제가 생긴다.

---
### 1.4.  일관성
> 모든 참조 값 x, y에 대해, x.equals(y)를 여러 번 반복해서 호출해도 항상 같은 값을 가짐 

두 객체가 같다면(수정되지 않는 한) 앞으로도 영원히 같아야 한다.

외부 요인 같은 equals 판단에 신뢰할 수 없는 자원이 끼어들어서는 안된다.

---
### 1.5.  null-아님
>모든 참조 값 x에 대해, `x.equals(null)`은 false

모든 객체가 null과 같지 않아야 한다.

## 2.  Equals 메서드 구현 방법 (+ 최적화)

### 2.1.  `==` 연산자를 사용해 자기 자신의 참조 인지 확인(최적화)
```java
if (this == obj) return true;
```
- **의미** : 두 객체가 완전히 동일한 인스턴스인지 확인 
- **효과** : 동일 참조일 경우, 아래의 복잡한 타입 검사나 필드 비교 생략 → **빠름**
	#단순_성능_최적화용
	  
### 2.2.  instanceof 로 타입검사  or getClass()로 비교
```java
if (!(obj instanceof MyClass)) return false;
if (o == null || getClass() != o.getClass()) return false;
```
- 연산자로 입력이 올바른 타입인지 확인
	- `getClass()`는 정확히 동일한 클래스만 비교하도록하기 때문에 추이성 보장이 쉽다 


- **입력을 올바른 타입으로 성공한다**. (2번을 했다면 이 과정은 100% 통과)
- 입력 객체와 자기 자신의 대응되는 **'핵심'필드들이 모두 일치하는지 하나씩 검사**  #중요필드만

완성본 
```java
@Override  
public boolean equals(Object o) {  
    // 1. 자기 자신이면 true    if (this == o) return true;  
    // 2 null 및 타입 체크   
if (o == null || getClass() != o.getClass()) return false;  
    // 3. 타입 캐스팅   
ColorPoint that = (ColorPoint) o;  
    // 4. 필드 비교   
return Objects.equals(point, that.point) && Objects.equals(color, that.color);
```


## 3.  번외 
### 3.1.  참고 : 컬렉션끼리 비교 시 

배열끼리 비교할 때는 단순히 `==`을 쓰면 같은 값을 가진 배열이라도 false가 나온다.
이럴 때는 Arrays.equals를 사용해라
```java
String[] words1 = {"a", "b", "c"};  
String[] words2 = {"a", "b", "c"};  
System.out.println(words1 == words2); // ❌ false  
  
System.out.println(Arrays.equals(words1, words2)); // ✅ true : 내용 비교 
```

```java
List<String> list1 = List.of("a", "b", "c");  
List<String> list2 = List.of("a", "b", "c");  
System.out.println("리스트 비교 단순 == : " + (list1 == list2)); // false  
  
// 해결 1. equals() 불러오기  
System.out.println("리스트 비교 equals() : " + list1.equals(list2)); // true  
  
// 해결 2. Objects.equals() 불러오기  
System.out.println("리스트 비교 Objects.equals() : " + Objects.equals(list1, list2)); // true
```

```text
리스트 비교 단순 == : false
리스트 비교 equals() : true
리스트 비교 Objects.equals() : true
```

>[!tip] 결론 : 컬렉션끼리 비교 시 내부 값을 기준으로 하고 싶다면 단순 `==`으로는 안된다.


### 3.2.  Equals 구현 후 질문할 것 

Equals를 구현했다면 이게 제대로 작동할지 생각해봐야 한다.
1. **대칭적인가❓**
2. **추이성이 있는가❓**
3. **일관적인가❓**


### 3.3.  Equals 파라미터로는 무조건 Object 타입 

> Object 외의 타입을 매개변수로 받는 equals 메서드는 선언하지 말자 

```java
❌ 잘못된 예시
public boolean equals(MyClass o){
}
```
- 이렇게 하면 Object.equals를 Override한 것이 아니라 Overloading한 것이다.
- 따라서, 타입을 구체적으로 명시한 equals를 구현하지마라

여기서 @Override 애노테이션 붙이면 컴파일 오류 뜰 것 



>[!tip] 참고로 equals를 개발자가 하나하나 정의하지 않는다. IDE or 라이브러리의 도움을 받을 수 있기 때문이다. 물론, 완벽한 것은 아니지만 사람처럼 실수를 저지를 가능성은 적다.


