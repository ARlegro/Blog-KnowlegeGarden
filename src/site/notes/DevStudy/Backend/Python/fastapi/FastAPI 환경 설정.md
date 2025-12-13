---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Python/fastapi/FastAPI 환경 설정/","noteIcon":"","created":"2025-12-03T14:52:50.524+09:00","updated":"2025-12-13T10:29:12.961+09:00"}
---



### 0.1.  가상환경에서 FASTAPI 설치


```c
pip install "fastapi[all]"
```
- FastAPI와 관련된 모든 종속성을 한 번에 설치 


*✅추가 - pydantic 설치* 
```c
pip install pydantic
```

### 0.2.  requirements.txt 

```c
pip install -r requirements.txt
```

### 0.3.  Miniconda 설치 

설치 스크립트 다운로드
```bsh
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda.sh
```

설치 진행 
```c
export PATH="$HOME/miniconda3/bin:$PATH"
// zsh용 초기화(hook) 적용 
conda init zsh

source ~/.zshrc
conda create -n fastapi-env python=3.10 -y
```

FastAPI & Uvicorn 설치
```c
// source ~/.zshrc
pip install fastapi 
pip install "uvicorn[standard]"
```

mongoDB
```C
pip install motor
```


### 0.4.  FAST API 실행해보기 
아래의 코드를 실행해보자
```PYTHON
from typing import Optional
from pydantic import BaseModel
from fastapi import FastAPI

# FastAPI instance 생성
app = FastAPI()

@app.get("/", summary="간단한 API", tags=['simple'])
async def root():
  return {"message":"Hello World"}


class Person:
  def __init__(self, name: str):
    self.name = ""
```

#### 0.4.1.  실행 명령어 - uvicorn
```c
// main = 파일명,  app = 파일 내 FastAPI인스턴스 
uvicorn main:app --port=8081 

uvicorn main:app --port=8081 --reload
```
*❓왜 uvicorn❓*
- FastAPI는 단독으로 실행되는게 아니다. 
- FastAPI는 프레임워크이기 때문에 이거를 띄어주려면 파이썬 기반의 비동기 웹서버를 사용해야 한다.
- FastAPI는 `uvicorn`이라는 **비동기 웹서버를 사용** 



