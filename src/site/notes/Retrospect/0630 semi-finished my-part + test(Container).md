---
{"dg-publish":true,"permalink":"/Retrospect/0630 semi-finished my-part + test(Container)/","noteIcon":"","created":"2025-06-30T23:18:55.341+09:00","updated":"2025-07-13T21:35:04.396+09:00"}
---




코드잇 조기퇴소 D-3 
오늘 오전 현재 진행 중인 팀프로젝트의 나의 파트는 끝났다.(테스트 코드 빼고)
한 일
- JWT Multi-Token 구현 
- CSRF 구현
- 강제로그아웃 구현
- AdminInitializer 구현 
- Auth 컨트롤러 테스트 코드 구현 

흠.... 총 4~5일 정도 걸렸는데 되게 빠르게 끝냈다.
원래는 더 빨리 끝내고 OAuth도 도전하고 마무리 해볼까?했지만 Multi-Token 방식을 구현해본적이 없어서 토요일에 공부좀하고 연습하다가 일요일부터 적용을 시작하느라 늦었다.

#### 컨트롤러 테스트 코드
이전에는 Service, Repository 단위 테스트를 공부하고 적용했던 적이 있었지만 Controller 부분의 단위테스트는 공부해본적이 없었다.
오늘 오전에 기능도 다 구현했으니 컨트롤러 테스트 코드를 공부했다.
공부 자료는 TDD 구현 잘 하시는 유튜버 "코딩하는 오후"님의 라이브 코딩과 Github 자료를 보면서 분석했다. 
몇 개 문법이랑 관련된 애노테이션만 익히면 되는거다보니 공부하고 적용하는데도 2시간도 안 걸린 것 같다. 테스트 코드 작성은 정말 귀찮고 아 당연한거를 왜 테스트하지?라는 생각을 항상 하는데 이번에 컨트롤러 Mock테스트하면서 나의 코드에 오류들이 몇 개 발견되는 것을 보고 '역시 테스트 코드는 중요하구나'라는 생각을 다시 하게 되었다.(몇 달 뒤에 또 이런 상황이 반복될 것 같지만 ㅎ)

와 근데... 코딩하는 오후님 라이브 코딩 영상 보면서 놀랐다.
TDD 라이브 코딩은 첨 보다보니 이게 TDD구나 하면서 놀랐다.

지금은 여유가 없으니 나중에 제대로 배울 기회가 생기면 도전해볼 가치가 있어보인다.

#### 테스트 컨테이너 도입 

기존 도커에 있는 Postgres의 데이터를 테스트 코드들이 훼손하는 것이 싫었다.
즉, 테스트 코드만의 독립적인 환경을 구축해보고자 이전에 Notion에 공부하고 정리했던 TestContainer를 정독하고 코딩하는-오후 님은 TestContainer를 어떻게 쓰나 보았다.
역시 이 분 유튜브를 보기 잘 했다. `@ServiceConnection`같은 최신 버전의 디테일들을 어떻게 적용하여 Config하고 활용하는지를 알게되었다.
확실히 편하고 TestConfig에 대해서 이해도가 생기면서 Security전용 Config도 만들며 Auth Controller의 단위테스트를 부사히 마칠 수 있었다.

Note
- Controller Mock 설정 자료 : [[DevStudy/Backend/Spring/test/WebMvcTest\|WebMvcTest]]
- TestContainer 설정 자료 : [[DevStudy/Backend/Spring/test/TestSecurityConfig\|TestSecurityConfig]] [[DevStudy/Backend/Spring/test/Use TestContainer\|Use TestContainer]]


