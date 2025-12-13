---
{"dg-publish":true,"permalink":"/DevStudy/Backend/RM/basic/Naming Convention/","noteIcon":"","created":"2025-12-03T14:52:50.073+09:00","updated":"2025-12-13T10:22:19.931+09:00"}
---



하....깃 커밋 충돌해결하다 이전에 진짜 자세히 적어놓은 파일이 사라졌어....
그래도 정리는 필요하니까 간단히.... gemini써서 하자 ... 하 진짜 다 날라갔네 



### Exchange 
- **`[서비스].[목적].[타입]`** 또는 **`[도메인].[이벤트].[종류]`** 형식으로 지정
- 예시: `order.events.direct`, `user.notifications.fanout`, `data.logs.topic`
- Exchange의 타입을 이름에 포함시키면 한눈에 라우팅 방식을 파악할 수 있다.


### Queue

- **`[서비스].[목적].[큐_타입]`** 또는 **`[도메인].[작업].[상태]`** 형식으로 지정합니다.
- 예시: `payment.processing.queue`, `email.sending.worker`, `user.signup.dlq` (DLQ: Dead-Letter Queue)
- **어떤 서비스가 이 큐를 사용**하고, **어떤 종류의 작업을 처리하는지** 명확하게 나타내는 것이 좋다.


### Routing Key (라우팅 키)
- **`[개체].[행동].[세부사항]`** 또는 **`[도메인].[이벤트].[상태]`** 형식으로 지정
- 예시: `user.created`, `product.updated.price`, `order.payment.failed`