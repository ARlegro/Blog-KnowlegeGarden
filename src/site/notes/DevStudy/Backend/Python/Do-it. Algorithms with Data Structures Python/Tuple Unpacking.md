---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Python/Do-it. Algorithms with Data Structures Python/Tuple Unpacking/","noteIcon":"","created":"2025-12-03T14:52:50.594+09:00","updated":"2025-12-13T10:29:37.968+09:00"}
---



### 튜플 언패킹 unlike 자바 
> 파이썬은 두 변수의 값을 동시에 교환하는 문법이 가능 (튜플 언패킹)

자바에서는 두 값을 교환할 때 temp변수를 만들었었다.
하지만 파이썬은 그럴 필요가 없다 (아래처럼)
```python
a = 5
b = 3
a, b = b, a  # a=3, b=5
```

**✅원리** 
![Pasted image 20250711111047.png](/img/user/supporter/image/Pasted%20image%2020250711111047.png)
1. Python은 내부적으로 (b, a)라는 튜플이 먼저 만들어진다.
2. 1번에서 만든 값을 `a, b` 에 할당한다
