---
{"dg-publish":true,"permalink":"/retrospect/0611-deep-dive-db/","noteIcon":"","created":"2025-06-16T21:24:01.417+09:00","updated":"2025-07-13T21:44:33.376+09:00"}
---



> DB를 공부하면서 느낀 점 

최근 2주 동안, DB 아키텍처, ACID, PostgreSQL 내부 작동 원리, 인덱싱 최적화, 동시성 제어 등 중급 수준의 DB 개념을 본격적으로 공부하면서 개발을 **DB 관점에서 바라보는 시야**를 갖게 되었다. 단순히 쿼리를 작성하는 수준을 넘어, 왜 그렇게 설계되어야 하는지, 어떤 방식으로 데이터가 읽히고 쓰이는지를 고민하게 된 것이다.

실제 프로젝트에서도 이를 직접 적용해볼 기회가 있었다. 여러 테이블에서 총 50M~100M 규모의 더미 데이터를 생성하고, 기존 팀원의 한방 쿼리와 내가 리팩토링한 쿼리를 비교해봤다. 그 결과는 놀라웠다.
(참고 링크 : [[DevStudy/DB/Fundamentals of DB engineering/이전 프로젝트 - 최적화 해보기\|이전 프로젝트 - 최적화 해보기]])
- 기존 쿼리: 20분 이상 무한 로딩
- 최적화 쿼리 (with 인덱싱): **125ms**
- 깃허브링크는 아직 미비 : 본가 컴퓨터에서 작업한건데 push를 안했었다... 귀찮아서... 나중에 크래프톤 일요일 휴무고 크래프톤 끝나기 전까지만 깃허브 시간날 때 꾸미자


"네카라쿠배 갈 것도 아닌데 DB를 그렇게까지 공부할 필요가 있냐?"는 말을 들은 적이 있다. 어느 정도는 맞고, 어느 정도는 틀리다고 생각한다.  
하지만, 나는 이번 경험을 통해 **DB를 이해하는 개발자야말로 진짜 실력을 가진 개발자**라는 확신을 가지게 됐다.
이제는 서버 로직이나 쿼리를 볼 때, '이게 DB 입장에서 효율적인가?'를 고민하게 되었고, 인덱싱 설계나 실행 계획을 분석하는 일도 낯설지 않다. PostgreSQL의 MVCC 구조나 동시성 처리 방식도 자연스럽게 떠올릴 수 있게 됐다. 최적화뿐 아니라, **엔지니어로서 시야가 확장**된 느낌이다.

---
개발을 시작한 지 4개월쯤 되었을 무렵, 부트캠프 팀 프로젝트에서 정말 뛰어난 실력자를 만난 적이 있다. 발전 속도나 내공이 압도적이었고, 동시성 이슈를 놓고 대화를 나누다 그분이 "read-read, read-write 상황 문제가 날텐데 Rock 경쟁을 어떻게 해소할 거냐"고 묻자 나는 아무 말도 하지 못했다. 지금의 나는 그 질문에 "PostgreSQL은 MVCC 기반이라서~"라고 자연스럽게 설명할 수 있다.  
그 분 같이 정말 뛰어난 개발자와 깊이 있는 토론하고 협업하려면 하려면 **ACID, 격리 수준, 동시성** 같은 개념은 반드시 알고 있어야 한다. 그걸 몰랐기에 토론이 안 됐었고, 알게 된 지금은 토론이 기다려진다.

--- 
예전에도 DB를 공부해보겠다고 PostgreSQL 책을 산 적이 있었다. 그러나 3일 만에 덮었고, 그 벽 앞에서 흥미를 잃은 적도 있다. 하지만 지금은 다르다. 벽을 넘었고, 오히려 DB가 재미있다.  
**그때 포기했던 DB가, 지금은 가장 보람찬 성장의 일부가 됐다.**
(DB가 재밌지만 DBA를 할게 아니기에... 적당히 공부하려고 한다 - 파티셔닝,샤딩 이런건...)