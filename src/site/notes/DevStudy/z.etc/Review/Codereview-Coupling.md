---
{"dg-publish":true,"permalink":"/DevStudy/z.etc/Review/Codereview-Coupling/","noteIcon":"","created":"2025-12-03T14:52:46.552+09:00","updated":"2025-12-09T17:19:48.996+09:00"}
---



[[DevStudy/z.etc/Fundamentals/정보처리기사/소프트웨어-개발/모듈\|모듈]]

정보처리기사를 공부하다가 객체 지향, 모듈에 관한 부분이 나와서 이전 프로젝트에서 해왔던 코드들을 되돌아 보는 시간을 가졌다.
결합도와 응집도 이 2가지 부분을 공부하면서 내 코드의 결함이 많이 느껴졌다.


### 0.1.  리팩토링 : 스탬프 결합도 ➡ 자료 결합도 
#### 0.1.1.  예시 1. 

**결합도가 존재하는 코드 - 스탬프 결홥도**
```java
public record ArticleInterestJdbc(  
    UUID id,  
    UUID articleId,  
    UUID interestId,  
    Instant createdAt) {  
  
  public static ArticleInterestJdbc create(Article article, Interest interest) {  
    UUID id = UUID.randomUUID();  
    return new ArticleInterestJdbc(  
        id,  
        article.getId(),  
        interest.getId(),  
        Instant.now()  
    );  
  }
```
이 코드는 객체를 파라미터로 통쨰로 전달한 코드이다.
create() 메서드는 Article 객체의 id, Interest객체의 id만들 사용하는데 객체 전체를 파라미터로 전달받았다.

이러한 형태의 결합도를 스탬프 결합도 라고 한다.
모듈의 6단계 결합도 중 2번쨰로 좋은 결합도이긴 하지만, 최고로 좋은 것은 아니다.
즉, 아직 더 리팩토링할 점이 있다는 것이다.

> 리팩토링 해보자 

**결합도를 없앤 코드 - 자료 결합도**
```java
public static ArticleInterestJdbc create(UUID articleId, UUID interestId) {  
  UUID id = UUID.randomUUID();  
  return new ArticleInterestJdbc(  
      id,  
      articleId,  
      interestId,  
      Instant.now()  
  );  
}
```
파라미터로 정말 필요한 데이터밖에 없다.
즉, **모듈간의 인터페이스가 자료 요소로만 구성**되게 함으로써, 다른 모듈에 영향을 주지 않는 결합도이다.
이는 재사용성, 확장성, 리팩토링 내성에 강한 코드로 만들어준다.


#### 0.1.2.  예시2. 

예시1 이랑 똑같은 경우다.
사실 객체를 전부 넘길 필요가 없다는 것을 알면서도 람다식(::)을 사용하고 싶어서 억지로 만들었었는데, 가독성보다 결합도를 낮추는게 더 중요한 것 같아서 바꾸는 것이 좋을 것 같다.

**스탬프 결합도** 
```java
  
  public static NotificationJdbc create(UnreadInterestArticleCount unreadInterestArticleCount) {  
    Instant createdAt = Instant.now();  
    return NotificationJdbc.builder()  
        .id(UUID.randomUUID())  
        .userId(unreadInterestArticleCount.getUser().getId())  
        .resourceId(unreadInterestArticleCount.getInterest().getId())  
        .resourceType(ResourceType.INTEREST)  
        .content(  
            unreadInterestArticleCount.getInterest().getName() + "와/과 관련된 기사가 "                + unreadInterestArticleCount.getArticleCount()  
                + "건 등록되었습니다.")  
        .createdAt(createdAt)  
        .updatedAt(createdAt)  
        .confirmed(false)  
        .build();  
  }  
}
```


**자료 결합도** 
```java
public static NotificationJdbc create(UUID userId, Interest interest, Long articleCount) {  
  Instant createdAt = Instant.now();  
  return NotificationJdbc.builder()  
      .id(UUID.randomUUID())  
      .userId(userId)  
      .resourceId(interest.getId())  
      .resourceType(ResourceType.INTEREST)  
      .content(  
          interest.getName() + "와/과 관련된 기사가 "              + articleCount  
              + "건 등록되었습니다.")  
      .createdAt(createdAt)  
      .updatedAt(createdAt)  
      .confirmed(false)  
      .build();  
}
```


### 0.2.  최악의 결합도 - 3달전의 나에게 

> 결합도의 6단계 중 최악은 `내용 결합도`  이다 `

이는, 한 모듈이 다른 모듈의 내부 구현, 자료를 직접 참조하는 것으로 아래와 같다.

```java
// 내부 필드에 직접 접근 (비추천)
class A {
    public int secret = 42;
}

class B {
    public void doSomething(A a) {
        a.secret = 999;  // A의 내부 구현에 직접 의존
    }
```
A의 구현이 바뀌면 B도 수정해야 됨
이는 유지보수에 최악 


