---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Java/Effective Java/Obect Create_Destroy(객체 생성과 파괴)/Private 생성자나 열거 타입으로 싱글톤임을 보장해라/","noteIcon":"","created":"2025-12-03T14:52:49.292+09:00","updated":"2025-12-13T10:46:59.477+09:00"}
---




> Private 생성자나 열거 타입으로 싱글톤임을 보장해라 

## 1.  싱글톤이란 ❓

**✅개념** 
- 애플리케이션 전체에서 오직 하나의 인스턴스만 존재하도록 보장하는 패턴

**❓언제 싱글톤 사용❓**
1. 상태를 공유할 필요가 없고
2. 매번 새로 생성할 필요가 없고 

> 보통 '유틸성 객체'일 가능성이 높다. 

이전에 내가 중급 프로젝트에서 구현한 것이 싱글톤 패턴 

## 2.  싱글톤 구현 방법

### 2.1.  private 생성자로  (feat. Only 자바)
>Spring을 쓰면 애초에 스프링이 관리하는 컨테이너의 빈이 싱글톤 객체라서 그냥 생성자 없이 `@Component` 붙이면 그게 싱글톤으로 관리된다.

지금은 그런 기능 없이 만들어 볼 것 

```java
public class MatchEngine {

    private static final MatchEngine INSTANCE = new MatchEngine();

		// 외부에서 생성자 접근 불가
    private MatchEngine() {
        
    }

    public static MatchEngine getInstance() {
        return INSTANCE;
    }

    public void match(Article a, Interest i) {
        // 필터링 로직
    }
```
 - private 생성자를 통해 외부에서 함부로 생성하지 못하도록 한다.
 - getInstance로 생성자를 조회할 수 있고 static 을 사용해서 오직 하나의 instance를 미리 생성해 놓을 수 있다.

### 2.2.  enum으로 만들기 ⭐⭐
>[!tip] Effective Java의 추천 
> Enum 방식은 리플렉션, 직렬화, 멀티스레드 환경에서도 깨지지 않는 싱글톤이 필요할 때 권장된다.

```JAVA
public enum MatchEngine{  
        INSTANCE;  
  
    private final Map<UUID, List<String>> caches = new HashMap<>();  
  
    public List<String> match(UUID uuid) {  
        if (!caches.containsKey(uuid)) {  
            return new ArrayList<>();  
        }  
  
        return caches.get(uuid);  
    }
```

다른 곳에서 사용 시 
```java
MatchEngine.INSTANCE.match(UUID.randomUUID());
```
> `MatchEngine.INSTANCE` 는 `enum MatchEngine` 타입의 값이자, 곧 `MatchEngine` 클래스의 싱글톤 객체

>[!tip] enum 상수는 항상 동일한 객체 ❗

| 방식                      | 설명                 | 장점            |
| ----------------------- | ------------------ | ------------- |
| private 생성자 + static 필드 | 가장 기본적인 싱글톤 구현     | 간단            |
| enum                    | Effective Java가 추천 | 역직렬화, 리플렉션 안전 |

1. **자바 언어 차원의 인스턴스 수 보장**
	- `ENUM`으로 만들면 JVM이 리플렉션이나 직렬화 과정에서도 추가 인스턴스를 만들 수 없게 설계된다.
	- 즉, **아주 복잡한 직렬화/리플렉션 공경에서도 제 2의 인스턴스가 생기는 일이 없어진다.**
	  
2. Thread-Safe
3. 간결하고 명확함 : private, static 사용 안 해도 된다.


>[!tip] 그러나 Spring의 @Component 사용하면 되기에 직접 ENUM 방식의 싱글톤을 구현할 일은 없다



## 3.  스레드 안정성 

앞서 싱글톤 객체를 만드는 2가지 방법에 대해 배웠다.
`ENUM`타입의 장점중 Thread-Safe는 `private()`방법에서도 가능은 하지만 주의 깊게 생각해야 한다.

### 3.1.  ENUM의 Thread-Safe

**ENUM의 상수 자체가 오직 하나의 객체**이다. 따라서 JVM에 클래스 로딩 시 딱 한 번만 생성되고, 직렬화/역직렬화 시 리플렉션에도 동일 객체가 유지된다.

즉, **아주 안정적인 싱글톤 패턴 구현 방식** 

### 3.2.  private() 의 Thread-Safe

**private 생성자 + `getInstance()` 방식**도 Thread-Safe하게 만들 수 있다.
바로, **Eager 초기화 방식일 때** 가능이다(static + final)

```java 
public class MatchEngine {
    private static final MatchEngine INSTANCE = new MatchEngine();

    private MatchEngine() { }

    public static MatchEngine getInstance() {
        return INSTANCE;
    }
```
- **미리 인스턴스를 만들어** 놓는 방식이다.
- 클래스 로딩 시점에 INSTATNCE가 초기화 된다.

> 따라서, 여러 스레드가 동시에 `getInstance()` 를 호출해도 항상 동일한 인스턴스를 반환 ➡ Thread-Safe (동기화 없이 안전)


### 💢문제가 되는 코드 
하지만 `private()` 방법 사용 시 Thread-Safe하지 않는 방법도 있다
바로, `Lazy 초기화`일 때이다.
```java
public class LazyMatchEngine {
    private static LazyMatchEngine instance;  // null로 시작

    private LazyMatchEngine() { }

    public static LazyMatchEngine getInstance() {
		    // race condition 발생 가능
        if (instance == null) {    
            instance = new LazyMatchEngine();  
        }
        return instance;
    }
```
- 앞선 예시와 달리 초기화 시점을 실제 호출 시점으로 미뤘다.
- 이 경우 동시성 제어를 하지 않으면 **두 개 이상의 스레드가 경합 시 두 번 생성** 될 수 있다 💢

> 따라서 Thread-Safe하지 않음 
