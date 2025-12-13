---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Redis/Setting/Redis 기본 명령어/","noteIcon":"","created":"2025-12-03T14:52:50.232+09:00","updated":"2025-12-13T10:24:11.149+09:00"}
---



### 접속 
```bash
redis-cli
```

### 정상 동작 확인
```bash
redis-cli 127.0.0.1:6379> ping
```


### 데이터(Key, Value) 저장 및 조회 



```bash 
[데이터 저장 set]
127.0.0.1:6379> set arlegro:name "lee y.h"
OK
127.0.0.1:6379> set arlegro:hobby soccer
OK

[데이터 조회 get]
127.0.0.1:6379> get arlegro:name
"lee y.h"

[저장된 모든 Key 조회- keys *]
127.0.0.1:6379> keys *
1) "arlegro:name"
2) "arlegro:hobby"

[삭제 - del [key 이름]]


```
>Note : 띄어쓰기가 된 값을 저장하려면 쌍따옴표(") 씀





>[!tip] Redis 네이밍 컨벤션 ➡ 콜론(:)을 사용해 계층적 표현 
>- `users:100:profile` : 사용자들(users) 중에서 PK가 100인 사용자(user)의 프로필(profile)
>- `products:123:details` : 상품들(products) 중에서 PK가 123인 상품(product)의 세부사항(details)
>- **컨벤션 장점**
>	1. **가독성** : 데이터의 의미와 용도를 쉽게 파악할 수 있다.
>	2. **일관성** : 컨벤션을 따름으로써 코드의 일관성이 높아지고 유지보수가 쉬워진다.
>	3. **검색 및 필터링 용이성** : 패턴 매칭을 사용해 특정 유형의 Key를 쉽게 찾을 수 있다 .
>	4. **확장성** : 서로 다른 Key와 이름이 겹쳐 충돌할 일이 적




### 데이터 저장 시 만료시간 정하기 
Redis는 RDBMS와는 다르게 데이터 저장 시 만료시간을 설정할 수 있다.
즉, 영구적으로 데이터를 저장하지 않고 일정 시간이 되면 데이터가 삭제되도록 Set 할 수 있다.

Redis 특성상 **메모리 공간이 한정** 되어 있기 때문에 **모든 데이터를 레디스에 저장할 수 없다.** 
따라서 **만료시간(TTL)을 활용**해 자주 사용하는 데이터만 레디스에 저장해놓고 쓰는 식으로 활용한다.

`set [key 이름] [value]
```bash
[set 키 value ex 시간]
127.0.0.1:6379> set arlegro:house proud ex 10
OK

[확인하기 - ttl [key이름]]
127.0.0.1:6379> ttl arlegro:house
(integer) 3

[만료시 - '-2'표시]
127.0.0.1:6379> ttl arlegro:house
(integer) -2
```


>[!tip] ttl 확인 시 '-1'표시는 애초에 만료시간을 설정하지 않은 것 


### 모든 데이터 삭제

```bash
flushall
```


