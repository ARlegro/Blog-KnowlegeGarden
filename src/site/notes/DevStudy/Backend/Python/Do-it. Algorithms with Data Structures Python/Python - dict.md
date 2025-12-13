---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Python/Do-it. Algorithms with Data Structures Python/Python - dict/","noteIcon":"","created":"2025-12-03T14:52:50.551+09:00","updated":"2025-12-13T10:29:24.431+09:00"}
---



> 파이썬은 Map<>의 형태가 없다. 대신, dictionary라는 자료구조를 사용한다
> - Java ➡ Map<K,Y>
> - Python ➡ dict[K, V]

### 0.1.  사용법 
#### 0.1.1.  기본 (생성, 삽입, 수정)
```PYTHON
my_dict = {}

my_dict["aba"] = 12
my_dict["b"] = 30
my_dict["c"] = 50
my_dict["c"] = 60

print(my_dict)
# {'aba': 12, 'b': 30, 'c': 60}
```

> 덮어쓰기로 수정이 된다.

#### 0.1.2.  조회 
```python
# 1. 대괄호 [ ] 접근법
print(my_dict["b"]) # 30
💢 # 주의 : key가 없으면 Error!!!

# 2. get() 접근법
print(my_dict.get("c")) # 70
# key가 없으면 'None' 반환

# 3. get() + default 방식 - 방어적 프로그래밍 ✅
print(my_dict.get("not_exit", "default_value"))
```

대괄호 접근법이 안 좋네 


#### 0.1.3.  삭제 
#del  #pop 
```python
my_dict = {}


my_dict["aba"] = 12
my_dict["b"] = 30
my_dict["c"] = 70

# ✅ 방법 1. del 
del my_dict['aba']
print(my_dict) # {'b': 30, 'c': 70}  << aba가 삭제됨 

# ✅ 방법 2. pop - 값 반환하면서 삭제 
popedB = my_dict.pop("b")
print(popedB) # 30    << b를 꺼내옴 
print(my_dict) # {'c': 70}   << b가 삭제됨 

# del my_dict['not_exist'] --> 없는 키라 에러 발생 💢
# my_dict.pop('not_exist') --> 없는 키라 에러 발생 💢
```






