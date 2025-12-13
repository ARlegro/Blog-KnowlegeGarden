---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Python/Annotated - 파이썬의 타입힌트 확장/","noteIcon":"","created":"2025-12-03T14:52:50.353+09:00","updated":"2025-12-13T10:25:46.220+09:00"}
---



## 1.  예전 타입 선언 방식과 문제 💢
```python
from fastapi import Query

q: list[str] = Query(default=None, max_length=50)
```

- `Query`라는 런타임 메타데이터가 타입 힌트 자리에 같이 붙어있었다.
- 이는, **정적 분석기 입장에서 헷갈려** 했음 

---
## 2.  Annotated 방식 등장 

> 목적 : 정적 타입과 추가 메타데이터를 분리

```python
Annotated[타입, 메타데이터...]
```
```python
from typing import Annotated
from fastapi import Query

q: Annotated[list[str], Query(max_length=50)]
```

이렇게 하면 정적 시간에 IDE/정적분석기에는 타입만 전달하고 런타임에는 메타데이터를 활용 


## 3.  권장하는 이유 
#타입힌트 #유효성검사 

- FastAPI같은 프레임워크 사용 시 **문서 만들고 검증할 때 메타데이터가 잘 전달됨**
