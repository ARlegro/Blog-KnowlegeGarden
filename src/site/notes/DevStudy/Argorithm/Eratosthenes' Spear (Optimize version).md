---
{"dg-publish":true,"permalink":"/DevStudy/Argorithm/Eratosthenes' Spear (Optimize version)/","noteIcon":"","created":"2025-12-03T14:52:51.942+09:00","updated":"2025-12-13T09:26:26.029+09:00"}
---

> [!success] 에라토스테네스의 채 (최적화 버전)


소수를 판별하는 알고리즘 
소수들을 대량으로 빠르고 정확하게 구할 수 있다 .


**원리**
1. **배열 생성 후 초기화** 
2. **2부터 시작해서 소수를 찾으면 그 배수들을 모두 지운다.**

--- 
### 최적화 에라토스테네스의 채 

```PYTHON
def sieve(x):
  is_primary_list = [True] * (x+1)
  for i in range(2, x+1):
    if is_primary_list[i]:
      for j in range(i*i, x+1, i):
        is_primary_list[j] = False
  return is_primary_list    

print(*sieve(10))
# True True True True False True False True False False False
#idx: 0 ~ 10
```



>[!QUESTION] 왜 P * P 부터 시작하는가 ❓
>- 임의의 배수 `m = p * k` 에서 k < p일 때
>- k가 소수라면 ➡ 이미 카운트
>- k가 소수가 아니라면 ➡ 이미 이전 숫자(p미만)에서 지워졌을 것이다.
>- 따라서 `p*2`가 아니라 `p*p`로 시작해도 충분하다 
>- 이미 `p*2, p*3, ~~~, p*(p-1)`은 더 작은 소수의 배수 or 소수로 카운팅 됨 

> `p*p`로 시작하면 중복 연산을 최소화하면서 올바르게 모든 배수를 제거 가능 


