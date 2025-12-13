---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Python/settings/Python 컨트롤러 확장/","noteIcon":"","created":"2025-12-03T14:52:50.437+09:00","updated":"2025-12-13T10:27:07.604+09:00"}
---


## 1.  옛날 버전
일단 컨트롤러 패키지를 만들
![Pasted image 20250910102424.png](/img/user/supporter/image/Pasted%20image%2020250910102424.png)어서 시작
> [!WARNING] 새 패키지 시작 시 `__init__.py`파일 생성 권장
> - **`__init__.py` 의미**
> 	- 이 디렉토리가 파이썬 패키지라는 것을 표시해줌. 
> 	- 즉, **모듈을 import하는 체계를 파이썬이 인식할 수 있도록** 함 
> 	- 추가 기능 : 패키지 import 시 실행할 초기화 코드 넣을 수 있다.
> - 최근 파이썬(3.3+)에서는 네임스페이스 패키지 덕분에 `__init.py__`없어도 import가 가능하다. But IDE 자동완성, 명시적 등의 이점을 얻고자 사용함
```python
from controller import items
```


### 1.1.  라우터 객체 생성

```python
from typing import Union
from fastapi import APIRouter

# router = FastAPI()
router = APIRouter(
  prefix="/items",
  tags=["items"],
  responses={404: {"description":"Not fount"}}
)

@router.get("/{item_id}")
def read_item(item_id: int, q: Union[str, None] = None):
  return {"item_id": item_id}
```

>[!question] FastAPI() 방식 vs APIRouter() 방식
>1. FastAPI 방식 : 애플리케이션 전체 대상 인스턴스
>2. APIRouter 방식 : 특정 기능(예: `/items`, `/users`)만 담당하는 작은 라우터 객체. (보통 main에서 이 router들을 모아 합침 `app.include_router(router)`)


### 1.2.  라우터 객체 main에서 import

```python
# main 클래스
from controller import items
from fastapi import FastAPI

app = FastAPI()

# include
app.include_router(items.router)
```

---
## 2.  최근 버전 - uv 

### 2.1.  Preview : UV 알아보기 
[[DevStudy/Backend/Python/UV - 패키지 관리 프로그램\|UV - 패키지 관리 프로그램]]


### 2.2.  세팅 

1. *FastAPI랑 실행기 설치 - `FastAPI`, `uvicron`*
	- `uv add fastapi "uvicorn[standard]"`
	  
2. *uvicorn.run 코드 세팅*
	```c
	def main():
		    uvicorn.run(app="app:app", host="0.0.0.0", port=8080, reload=True)
		    # app:app 의미 : app.py라는 파일에서 app이라는 모듈을 참고하겠다
	```
	- `uvicorn.run`
		- 실제 브라우저 요청을 죽받는 Uvicorn서버를 띄어서 앱을 실행하겠다는 뜻
	- `app:app` - "파일명:변수명"
		- 파일명 예시 = app.py (py 생략)
		- 변수명 : app.py안에 정의된 ~객체 
			```c
			# app.py
			from fastapi import FastAPI
			app = FastAPI()
			```
	- *host*
		- 0.0.0.0 : 모든 네트워크 인터페이스에서 접근 가능 
		- 127.0.0.1 : 로컬 PC에서만 접근 가능 

3. *설정 파일* 
	- 표준 경로 `config/config.py`
