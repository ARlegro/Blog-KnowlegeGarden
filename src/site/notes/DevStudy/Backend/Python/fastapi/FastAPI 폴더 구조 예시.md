---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Python/fastapi/FastAPI 폴더 구조 예시/","noteIcon":"","created":"2025-12-03T14:52:50.533+09:00","updated":"2025-12-13T10:29:08.106+09:00"}
---



## 1.  폴더 구조 1 
```java
app/
├── main.py                     # FastAPI 애플리케이션 엔트리포인트
├── core/                       # 설정, 보안, 유틸성 모듈
│   ├── config.py               # 환경 변수 로드, 전역 설정
│   └── security.py             # 인증, JWT 로직 등
├── db/
│   ├── base.py                 # Base = declarative_base() 등
│   ├── session.py              # DB 연결 엔진, 세션 생성
│   └── migrations/             # Alembic 마이그레이션 폴더
├── models/                     # SQLAlchemy 모델 정의
│   ├── user.py
│   ├── item.py
│   └── ...
├── schemas/                    # Pydantic 스키마
│   ├── user.py
│   ├── item.py
│   └── ...
├── crud/                       # DB 처리 로직 (Create, Read, Update, Delete)
│   ├── user.py
│   ├── item.py
│   └── ...
├── api/
│   └── v1/                     # 버전별 API (v1, v2 등)
│       ├── endpoints/          # 실제 라우트(엔드포인트)들을 모아둔 디렉토리
│       │   ├── user.py
│       │   ├── item.py
│       │   └── ...
│       └── routers.py          # v1 라우터들을 모아 FastAPI에 등록하는 모듈
└── tests/                      # 테스트 코드
    ├── test_user.py
    ├── test_item.py
    └── ...
```

## 2.  폴더 구조 2 
FastAPI에서는 대체로 다음과 같은 구조를 많이 사용.
```pgsql 
app/
 ├── main.py
 ├── api/
 │    ├── routes/
 │    │     └── users.py
 │    └── dependencies/
 │          └── auth.py
 ├── core/
 │    ├── config.py
 │    └── security.py
 ├── models/
 │    └── user.py
 ├── schemas/
 │    └── user.py
 ├── services/
 │    └── user_service.py
 ├── db/
 │    ├── base.py
 │    └── session.py
 └── tests/

```
- **`models/`**: SQLAlchemy ORM 모델 정의 (DB 테이블 구조)
- **`schemas/`**: Pydantic 기반 요청/응답 모델 정의 (NestJS의 DTO 역할)
- **`api/routes/`**: 라우터 (Nest의 Controller 역할)
- **`services/`**: 비즈니스 로직 (Nest의 Service 역할)
- **`core/`**: 설정, 보안, 유틸성 코드 등
- **`db/`**: 데이터베이스 연결, 세션 관리

## 3.  라우팅 전략 
참고 : [공식 문서](https://fastapi.tiangolo.com/ko/tutorial/bigger-applications/#import-apirouter)

```java
.
├── app                  # "app" is a Python package
│   ├── __init__.py      # this file makes "app" a "Python package"
│   ├── main.py          # "main" module, e.g. import app.main
│   ├── dependencies.py  # "dependencies" module, e.g. import app.dependencies
│   └── routers          # "routers" is a "Python subpackage"
│   │   ├── __init__.py  # makes "routers" a "Python subpackage"
│   │   ├── items.py     # "items" submodule, e.g. import app.routers.items
│   │   └── users.py     # "users" submodule, e.g. import app.routers.users
│   └── internal         # "internal" is a "Python subpackage"
│       ├── __init__.py  # makes "internal" a "Python subpackage"
│       └── admin.py     # "admin" submodule, e.g. import app.internal.admin
```

> include_router로 여러 엔드포인트를 일괄 등록 가능하다.

프로젝트 규모가 커질 경우 하나의 엔트리 포인트(`main.py`)에 모든 라우트를 정의할 수 없고 여러 파일로 분리해야 한다. 
위의 폴더 구조 예시는 각각의 APIRouter들(endpoints내부)을 `routers.py`의 APIRouter에 등록한 뒤 `main.py`의 라우터에 연결하는 구조이다.

>[!tip] APIRouter객체는 "mini FastAPI Class"로 생각하면 된다

```python
from fastapi import APIRouter, FastAPI, UploadFile

audio_router = APIRouter()  

@audio_router.post("/transcribe")
async def transcribe(file: UploadFile):
    print("")
```

```python
# routers.py
## 라우터들을 모아 FastAPI에 등록하는 모듈
from fastapi import APIRouter, FastAPI
from router.endpoints.audio import audio_router 
# ⭐ 이거 main.py의 경로 기준으로 from해야 함 

api_router = APIRouter()
api_router.include_router(audio_router, prefix="/audio")
```

```python
# main.py
from fastapi import FastAPI
import uvicorn
from pydantic import BaseModel
from router.routers import api_router

app = FastAPI()
app.include_router(api_router)

@app.get("/")
def main():
    print("Hello from summarization!")
    return "hello"  

if __name__ == "__main__":
    uvicorn.run(app="main:app", host="0.0.0.0", port=8088, reload=True)
```



---
