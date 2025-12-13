---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Java/Effective Java/Obect Create_Destroy(객체 생성과 파괴)/파라미터가 많다면 Builder를 고려해라/","noteIcon":"","created":"2025-12-03T14:52:49.257+09:00","updated":"2025-12-13T10:52:26.198+09:00"}
---



> 정적 팩토리 메서드를 만들던 생성자를 만들던 '많은 매개변수'앞에서는 대응하기 어렵다.

## 1.  많은 매개변수 생성과 문제점 - Builder 패턴 필요성 
```java

public Notification(int count, int size, Customer customer, Article article, int page, int likes)
```
옛날에는 위처럼 모든 필드를 **생성자에 순서대로 나열**하여 객체를 생성하곤 했다. 하지만 매개변수가 많아질 수록 아래와 같은 문제가 발생한다💢
1. **순서 헷갈림** : 순서 잘못 넣고, 컴파일/런타임 오류 발생
2. **의도 파악 어려움** 

❓점층적 생성자 방식은 어떤가❓
```java 
public Notification(int count, int size, Customer customer, Article article) {
		new Notification(count, size, customer, article, 0, 10)
}

public Notification(int count, int size, Customer customer, Article article, int page, int likes)
```
선택적 매개변수를 처리하기 위해 오버로딩된 생성자를 계속 추가하면, **코드가 장황해지고 관리가 어려워진다.**


**❓Setter방식은 어떤가❓** #불변성_침해  #일관성_침해
모든 필드에 대해 `setter`를 호출하는 방식은, 객체의 불변성과 일관성을 쉽게 무너뜨린다 Cuz 객체가 완성되기 전에 중간 상태 노출 + 변경 

## 2.  Builder 패턴과 장점 
> Builder 패턴은 위의 문제들을 해결해준다.

1. **가독성** : 어떤 값이 어떤 필드에 들어가는지 명확함
2. **유연성** : 선택적 파라미터 가능
3. **일관성 보장** : 필수값만 채우고 기본값은 그대로 둘 수 있음 
4. **FluentAPI** : 가독성 Good 


## 3.  사용법 
> 단순히, 자바만 쓰면 클래스 내부에 Builder static Class를 둬서 길게 만들어야 한다.
> But Spring의 Lombok 사용 시 손쉽게 구현할 수 있다.

### 3.1.  기본 
```java
@Builder  
public class Notification {  
    private int count;  
    private int size;  
    private Integer page;  
    private int likes;  
}
```

```java
Notification build = Notification.builder()  
        .likes(1)  
        .count(2)  
        .page(3)  
        .size(4)  
        .build();

// Notification(count=2, size=4, page=3, likes=1)		
```


### 3.2.  Default

만약 builder에서 특정 필드 값을 부여하지 않았다면??
예를 들어, 위의 예에서 page를 누락했다면❓ `null` or 0이 될 수 있다.

```java
Notification build = Notification.builder()  
        .likes(1)  
        .count(2)  
        .build();

// Notification(count=2, size=0, page=null,✅ likes=1) 
```

>[!tip] 만약 기본적으로 원하는 값이 있다면 @Builder.Default 를 이용해라
```java
@Builder  
@ToString  
public class Notification {  
    private int count;  
    private int size;  
  
    @Builder.Default  
    private Integer page = 12;  
    private int likes;
```

> [!WARNING] @Builder.Default  은 final 필드에 못 쓴다.
> @Builder.Default 는 필드기반으로 빌더 코드를 생성하기 때문에 final 키워드를 못 쓴다.


### 3.3.  특정 필드 필수로 만들기 

> 만약, 특정 필드가 필요하다면??? 

1. `@NotNull` 사용 
2. 생성자 + `@Builder` 사용 
	 - 생성자를 만들고 `@Builder`를 붙여주면 그 생성자의 파라미터는 빌더시 반드시 포함해야 한다.
	```java
	private Integer page;  
	private int likes;  
	  
	@Builder  
	private Notification(int size, Integer page) {  
	    this.size = size;  
	    this.page = page;  
	}
	```
	- 이렇게 하면, builder 시 page, size 누락 시 오류가 발생한다.


## 4.  Builder 패턴 단점 💢
성능 : Builder 생성 비용이 크지는 않지만 그래도 성능에 민감한 상황에서는 문제가 될 수 있다.

>[!EXAMPLE] 빌더 패턴 고려하기 전 
> - 애초에 그렇게 파라미터가 많이 필요하다는거는 객체의 책임이 많다는 것일수도 있다.(SRP 위반) 
> - 객체지향 생활체조 원칙에 의하면 필드는 3개가 적당(반드시는 아님). 따라서 설계를 다시 하는 것도 고려할 수 있다.


## 5.  언제 사용 ❓

> 처리해야 할 매개변수가 많다면 빌더 패턴을 선택하는 것이 좋다.


