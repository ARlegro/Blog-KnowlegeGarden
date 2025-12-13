---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Python/Do-it. Algorithms with Data Structures Python/Variable Value in python/","noteIcon":"","created":"2025-12-03T14:52:50.583+09:00","updated":"2025-12-13T10:29:32.591+09:00"}
---



> 파이썬의 변수는 값을 갖지 않는다.

### 모든 것은 객체(Object) 취급 

- 파이썬에서는 데이터, 함수, 클래스, 모듈, 패키지 등을 모두 객체로 취급한다.
- 객체는 자료형이며 메모리를 차지한다.


```python
data = 3
print(id(data)) # 11760744 <<< 식별 번호 
```
- data에 단순히 int값을 넣어줬는데 마치 JAVA에서 객체의 주소마냥 print됐다.
- 이는 파이썬이 자료형/기본형 구분 없이 전부 객체 처리한다는 것 
		![Pasted image 20250711120036.png](/img/user/supporter/image/Pasted%20image%2020250711120036.png)

```python
data = 3
x = 3

print(id(data)) # 11760744
print(id(x)) # 11760744
#같은 식별번호
```

