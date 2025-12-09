---
{"dg-publish":true,"permalink":"/DevStudy/Argorithm/python/List Comprehension/","noteIcon":"","created":"2025-12-03T14:52:52.407+09:00","updated":"2025-12-09T17:19:42.748+09:00"}
---



> 리스트를 쉽게, 짧게 한 줄로 만들 수 있는 파이썬 문법 

자바에서는 배열, 리스트를 만들 때 최소 3~4줄의 공간이 필요하다 Cuz 선언과 할당을 순차적으로하기 때문 

파이썬은 이러한 문제를 해결하여 **"짧게 한 줄로"** 가능하게 했다
기본적인 구조는 아래와 같다
```python
[expr for 변수 in iterable]

arr = [i * 2 for i in range(3)]
# [0, 2, 4]
```
- `expr` ➡ 배열의 담을 값을 기술


**✅추가 : 조건문 추가** 

```python
[expr for 변수 in iterable if 조건]

arr = [n for n in range(1, 11) if n % 2 == 0]
```
- 맨 뒤에 if문을 사용해서 해당 조건문에서 참이 나온 값만 배열의 원소가 되도록 필터링 
