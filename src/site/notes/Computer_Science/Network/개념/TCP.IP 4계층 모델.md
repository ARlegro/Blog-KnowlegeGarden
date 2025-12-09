---
{"dg-publish":true,"permalink":"/Computer_Science/Network/개념/TCP.IP 4계층 모델/","noteIcon":"","created":"2025-12-03T14:52:46.189+09:00","updated":"2025-12-09T17:19:41.977+09:00"}
---



Prev : [[Computer_Science/Network/개념/OSI 7 Layer\|OSI 7 Layer]]


---
# TCP/IP 4Layer 

### OSI 7Layer의 진화 버전 
- 💢*OSI 7계층 모델*은 이미 시장에서 **도태된 네트워크 시스템**이라고도 함 
- ✅현대에는 OSI 7계층 모델의 일부를 계승한 **TCP/IP 모델을 대부분 사용**
- 또한, TCP/IP 모델에서도 한번 업데이트가 이루어져 사실상 **Updated TCP/IP Model이라 불리는 모델**을 사용

![Pasted image 20250829024525.png](/img/user/supporter/image/Pasted%20image%2020250829024525.png)
[출처(클릭)](https://dydgustmdfl1231.tistory.com/54)


###  계층 차이 (vs OSI 7 Layer)
![Pasted image 20250829105232.png](/img/user/supporter/image/Pasted%20image%2020250829105232.png)

![Pasted image 20250829112946.png](/img/user/supporter/image/Pasted%20image%2020250829112946.png)
(출처 : [인파](https://inpa.tistory.com/entry/WEB-%F0%9F%8C%90-TCP-IP-%EC%A0%95%EB%A6%AC-%F0%9F%91%AB%F0%9F%8F%BD-TCP-IP-4%EA%B3%84%EC%B8%B5))

---
## 계층 1 - 네트워크  계층 
TPC/IP 1계층 = OSI의 1계층(물리) + 2계층(데이터링크)

> Network, H/W와 직접적으로 연결되어 **데이터를 물리적으로 전송** 

- 대표적 기술
	- Wi-Fi
	- 이더넷 : [[Computer_Science/Network/개념/클라이언트-서버 프로그래밍 모델\|클라이언트-서버 프로그래밍 모델]]
	- ARP : IP주소를 MAC주소로 매핑 (Address Resolution Protocol)

---
## 계층 2 - 네트워크 계층 
> 핵심 역할 : IP 주소를 활용하여 데이터를 목적지까지 전달

내용 : [[Computer_Science/Network/개념/OSI 7 Layer\|OSI 7 Layer]] -> 3계층 

---
## 계층 3 - 전송 계층 
> 신뢰성 보장의 핵심 

OSI 7 Layer의 4계층과 동일 

자세한 내용 : [[Computer_Science/Network/개념/OSI 7 Layer\|OSI 7 Layer]] -> 4계층)

- Port를 사용해서 애플리케이션을 찾는다
- *대표적 프로토콜* : TCP, UDP, RTP, RTCP 

---
## 계층 4 - 응용 계층 

OSI 모델의 세션, 표현, 응용 계층을 합친 것 


---
## 동작 순서

출처: [https://inpa.tistory.com/entry/WEB-🌐-TCP-IP-정리-👫🏽-TCP-IP-4계층](https://inpa.tistory.com/entry/WEB-%F0%9F%8C%90-TCP-IP-%EC%A0%95%EB%A6%AC-%F0%9F%91%AB%F0%9F%8F%BD-TCP-IP-4%EA%B3%84%EC%B8%B5) [Inpa Dev 👨‍💻:티스토리]

![Pasted image 20250829114059.png](/img/user/supporter/image/Pasted%20image%2020250829114059.png)
1. 송신측 클라이언트의 **애플리케이션 계층**에서 어느 웹 페이지를 보고 싶다라는 HTTP 요청을 지시한다.
2. **전송 계층**에서는 애플리케이션 계층에서 받은 데이터(HTTP 메시지)를 통신하기 쉽게 조각내어 안내 번호와 포트 번호(TCP 패킷)를 붙여 **네트워크 계층에 전달**한다.
3. **네트워크 계층**에서 데이터에 **IP 패킷을 추가해서 링크 계층에 전달**한다.
4. 링크 계층에서는 수신지 MAC 주소와 이더넷 프레임을 추가한다. 
5. 이로써 네트워크를 통해 송신할 준비가 되었다.
6. 수신측 서버는 링크 계층에서 데이터를 받아들여 순서대로 위의 계층에 전달하여 애플리케이션 계층까지 도달한다.
7. 수신측 애플리케이션 계층에 도달하게 되면 클라이언트가 발신했던 HTTP 리퀘스트를 수신할 수 있다.

