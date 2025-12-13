---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Java/Effective Java/Obect Create_Destroy(객체 생성과 파괴)/인스턴스화를 막으려거든 private 생성자를 사용해라/","noteIcon":"","created":"2025-12-03T14:52:49.269+09:00","updated":"2025-12-13T10:53:04.854+09:00"}
---



> 인스턴스화를 막으려거든 private 생성자를 사용해라


>[!tip] 생성자를 명시하지 않아도, 컴파일러가 자동으로 기본 생성자를 만들어준다. 

```java
public class Test {   
    private String name;  
}
```

```java
Test test = new Test(); ✅ 이게 가능 
```

사용자는 이게 의도된 것인지 아닌지 구분할 수 없다.
코드 내에서 뿐만 아니라 java doc 같은 코드 외부 문서에서도 타인이 볼 때도 마찬가지다.

## 1.  해결방법 
> 이러한 상황을 막기 위해 **private 생성자를 추가**하면 클래스의 인스턴스화를 막을 수 있다.


## 2.  💢 단점 
> 상속 불가능 
- 모든 생성자는 상위 클래스 생성자를 호출해야하는데 private으로 선언되면 하위 클래스가 상위 클래스의 생성자에 접근할 길이 막혀버린다.


>[!tip] 근데 JPA 사용하면 기본 생성자는 있어야 하므로 Protected로 ㄱ 
>- JPA는 프록시 객체 생성할 때, 내부적으로 기본 생성자 사용
>- JPA는 리플렉션 사용해서 인스턴스를 생성하는 기본 생성자가 필수