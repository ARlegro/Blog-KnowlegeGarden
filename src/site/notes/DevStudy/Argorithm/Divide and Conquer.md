---
{"dg-publish":true,"permalink":"/dev-study/argorithm/divide-and-conquer/","noteIcon":"","created":"2025-07-17T17:18:13.525+09:00","updated":"2025-08-01T00:05:46.253+09:00"}
---

#분할정복

> 그대로 해결할 수 없는 문제를 작은 문제로 분할하여 문제를 해결하는 방법
> ex. **퀵 정렬**, **병합 정렬**, 이진 탐색, 선택 문제?


### 분할 정복 예시 

#### 예시 1. C의 N제곱 구하기 

❌분할 정복 안 할 시 
```PYTHON
  
C = 2
m = 10
def recursive_power(C, count):
  if count == m:
    return 1
    
  return C * recursive_power(C, count + 1)

result = recursive_power(C, 0)
print(f'{result}')
```


✅ 분할 정복 시 
```python
def recursive_power(C, n):
  if n == 1:
    return C

  if n % 2 == 0:
    y = recursive_power(C, n / 2)
    return y * y
    
  else:
    y = recursive_power(C, (n-1) / 2)
    return y * y * C
    
C = 2
n = 10
result = recursive_power(C, n)
print(f'{result}')
```
> 기존 O(n)의 시간복잡도보다 더 빠르게 가능 




