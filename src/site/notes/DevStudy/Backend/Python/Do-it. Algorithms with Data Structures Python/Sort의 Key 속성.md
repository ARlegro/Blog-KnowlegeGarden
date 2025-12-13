---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Python/Do-it. Algorithms with Data Structures Python/Sort의 Key 속성/","noteIcon":"","created":"2025-12-03T14:52:50.571+09:00","updated":"2025-12-13T10:29:40.914+09:00"}
---



`sort()` or `sorted()` 함수의 **`key` 속성은 정렬 기준을 지정하는 데 사용**된다.
- key 매개변수에 **함수를 전달**하여 각 요소에 대한 정렬 키를 계산 
- key 매개변수에 여러 요소 전달 시 ➡ 앞쪽 기준을 우선으로 정렬하고 그 뒤가 우선 
	- ex. `arr.sort(key=lamda : (x[0]), x[1]))` ➡ 첫 번째 원소를 기준으로 정렬하고 같으면 두 번째 원소 기준으로 정렬 

>[!tip] sort vs sorted()
>- sort : 원본 리스트 변경
>- sorted() : 새로운 정렬된 리스트 반환 (원본 리스트는 변경❌)

## 사용 예시 

### value값의 길이를 기준으로 정렬 시 

```python
words = ["apple", "banana", "kiwi", "orange"]
words.sort(key=len) # () 안 쓰는게 key point -> 람다식에서도 바로 실행안시키는 그런거임
```

### 람다식 -  특정 key 값을 기준으로 

#오름차순  #내림차순 
```python
my_list = [{'name': 'Alice', 'age': 30}, {'name': 'Bob', 'age': 25}, {'name': 'Charlie', 'age': 35}]

# 오름차순 
my_list.sort(key=lambda x: x['age'])

#내림차순 ( - )
my_list.sort(key=lambda x: -x['age'])
```


