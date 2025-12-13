---
{"dg-publish":true,"permalink":"/DevStudy/Backend/saga-pattern/Intro. saga-pattern/","noteIcon":"","created":"2025-12-03T14:52:49.877+09:00","updated":"2025-12-13T10:19:37.224+09:00"}
---


### 0.1.  Saga 패턴 

> MSA에서 분산된 트랜잭션을 관리하는 전략 중 하나 

긴 Transaction을 **작은 로컬 트랜잭션들로 나눠서 처리**하고, 문제가 생기면 **보상 작업(compensation)** 을 통해 이전 상태로 되돌리는 방식

하나의 전역 트랜잭션 대신, 워크플로우를 일련의 로컬 트랜잭션으로 분할하며, 각 로컬 트랜잭션은 개별 서비스에 의해 관리된다.

>[!tip] Saga는 '롤백'이 아니라 '보상'을 통해 문제를 처리한다.


Lock이나 조정없이 '최종 일관성'을 유지하는 것 

### 0.2.  예시: 주문 -> 결제 -> 재고 -> 배송
1. **주문(Order)** 서비스가 "OrderCreated" 이벤트를 발행
2. **결제(Payment)** 서비스는 이 이벤트를 구독하고 결제를 수행, 성공 시 "PaymentCompleted" 이벤트 발행
3. **재고(Inventory)** 서비스가 재고를 차감하고 "InventoryUpdated" 이벤트 발행
4. **배송(Shipping)** 서비스가 최종 처리

만약 재고가 부족하면 ❓
- "InventoryFailed" 이벤트를 발행하고,
- 이 이벤트를 받은 **각 서비스가 보상 로직** 실행 (결제 취소, 주문 취소 등)


## 1.  간단히 구현해보기

https://mobisoftinfotech.com/resources/blog/microservices/saga-pattern-spring-boot-rabbitmq-tutorial


### 1.1.  개요 

간단한 송금 MicroService 시스템 구현 
1. 계좌의 차변 서비스 (8081 포트)
	- `post /transfer` ➡ `TransferRequested` 이벤트를 발행
	- `GET /accounts` ➡ 인메모리 잔고를 확인 
	  
2. 계좌의 대변 시스템 (8082 포트)
	- `GET /accounts` ➡ 인메모리 잔고 확인 

RabbitMQ 사용 예정 

### 1.2.  진행 Flow 

1. **차변 서비스**가 `TransferRequested` 이벤트를 소비
2. **대변 서비스**가 `DebitCompleted` 이벤트를 소비
	- 대변(입금) 로직이 성공하면, "받는 계좌(`toAccount`)"에 대변(입금)하고, `CreditCompleted` 이벤트를 발행
	- 대변(입금) 로직이 실패하면, `CreditFailed` 이벤트를 발행
3. **차변 서비스**는 또한 `CreditFailed` 이벤트를 수신
	- 보상 환불 발행 
	- "보내는 계좌(`fromAccount`)"에 금액을 다시 추가

> 기본적인 @Transaction 한 곳에 몰아 넣으면 당연히 할 수 있는 것이다.
> 하지만 RabbitMQ같은 구조라면??? 이를 Saga-Pattern으로 해결해 볼 것 


