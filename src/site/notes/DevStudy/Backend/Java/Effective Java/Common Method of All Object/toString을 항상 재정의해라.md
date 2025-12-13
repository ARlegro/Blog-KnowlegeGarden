---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Java/Effective Java/Common Method of All Object/toString을 항상 재정의해라/","noteIcon":"","created":"2025-12-03T14:52:49.227+09:00","updated":"2025-12-13T10:42:06.199+09:00"}
---


> toString을 항상 재정의해라 

toString을 재정의하지 않았을 때는 단순히 `클래스명_16진수로 표시한 해시코드`를 출력할 것이다.
```java
System.out.println(dog);
// 결과 : Dog@4eec7777
```

> [!tip] toString의 일반 규약
> 간결하면서 사람이 읽기 쉬운 형태의 유익한 정보를 반환할 것 

## 1.  잘 정의된 toString의 장점 

- 사용감이 좋다
- 디버깅하기 좋다 : 쓸모 있는 내용만 출력해주므로 


> [!tip] toString은 그 객체가 가진 주요 정보를 모두 반환하는 것이 좋다
> - 단순히 아무 필드의 정보를 반환하도록 하는 것은 좋지 않다.
> - Cuz 디버깅 시 알아차리기 힘들다.
> - 따라서, 중요 정보를 문서화해서 toString 재정의 하는 것이 좋음 
> - 💢But, 포맷을 한다는 것은 변경에 자유롭지 못 할 것이다. 그 포맷에 얽매여서 객체를 만들고 저장해야 하기 때문이다. 따라서 포맷을 반드시 만드는 것은 아니더라도 의도는 명확히 밝혀야 한다.












