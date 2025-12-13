---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Redis/Performance Comparison Based Caching/","noteIcon":"","created":"2025-12-03T14:52:50.275+09:00","updated":"2025-12-13T10:23:16.233+09:00"}
---



## 1.  테스트 

>100만개의 더미데이터를 생성 후 조회하는 로직

### 1.1.  첫 호출 : 캐싱 전 - 813ms 
>이 때는, 캐싱이 안되어 있기에 느리다
![Pasted image 20250604231025.png](/img/user/supporter/image/Pasted%20image%2020250604231025.png)




### 1.2.  캐싱 후 조회 
#### 1.2.1.  2번째 조회 - 39ms

![Pasted image 20250604231034.png](/img/user/supporter/image/Pasted%20image%2020250604231034.png)



#### 1.2.2.  3번째 이후부터 - 6ms 
![Pasted image 20250604230613.png](/img/user/supporter/image/Pasted%20image%2020250604230613.png)


### 1.3.  결론 

813ms ➡ 6ms
무려 130배 차이가 난다.
물론 Cache Aside전략을 쓰기에 Redis 도입으로 인해 첫 조회 시 불필요한 조회 시간이 들긴 했지만 그 이후부터는 엄청나게 효과적이다.


