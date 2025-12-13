---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Python/pydantic/Pydantic/","noteIcon":"","created":"2025-12-03T14:52:50.646+09:00","updated":"2025-12-13T10:27:51.808+09:00"}
---


패키지 설치 
```bash
pip install pydantic
```

### 0.1.  사용법 


#### 0.1.1.  클래스 정의 
Pydantic 모델을 생성하려면, 먼저 **`BaseModel` 클래스를 상속하는 클래스**를 정의 必
클래스 내부에서 모델의 필드를 클래스 변수로 정의
```python
from pydantic import BaseModel

class User(BaseModel):
    name: str
    email: str
    account_id: int

class User(BaseModel): 
		name: str 
		email: EmailStr # EmailStr로 타입 변경 
		account_id: int
```

#### 0.1.2.  생성 
```python
user = User(name="Alice", email="alice@example.com", account_id=12345)
```


#### 0.1.3.  생성 시 예외 처리 
만약 잘못된 타입으로 객체를 생성하려고하면 에러가 터진다
아래의 코드는 int형태가 아닌 문자열로 파라미터를 전달해서 터지는 예외이다.

```python
# 잘못된 타입으로 객체 생성 시도
try:
    user_invalid = User(name="Bob", email="bob@example.com", account_id="invalid") 
except Exception as e:
    print(e)
# 출력: 1 validation error for User... value is not a valid integer
```


#### 0.1.4.  유효성 검사 설정 

```python
from pydantic import BaseModel, validator

class User(BaseModel):
    name: str
    email: str
    account_id: int

    @validator('account_id') # account_id 필드에 대한 유효성 검사기
    def account_id_must_be_positive(cls, value):
        if value <= 0:
            raise ValueError('account ID must be positive')
        return value

# 예시: 음수 값 시도
try:
    user_negative_id = User(name="Eve", email="eve@example.com", account_id=-50)
except Exception as e:
    print(e)
# 출력: 1 validation error for User... account ID must be positive
```


#### 0.1.5.  객체 JSON or 딕셔너리로 변환 

> `.json()`메서드만 호출하면 된다.

```python
user = User(name="Frank", email="frank@example.com", account_id=98765)
json_string = user.json()
print(json_string) # {"name": "Frank", "email": "frank@example.com", "account_id": 98765}
```

> `.dict()`메서드 호출 ➡ 딕셔너리 객체 
```python
python_dict = user.dict()
print(python_dict) # {'name': 'Frank', 'email': 'frank@example.com', 'account_id': 98765}
```


### 0.2.  Dict 타입

```python
from pydantic import BaseModel
from typing import Dict, Any

class ProfileDTO(BaseModel):
    userId: str
    introduction: str
    data: Dict[str, Any]  # 어떤 key-value든 허용

# 사용 예시
profile = ProfileDTO(
    userId="user123",
    introduction="안녕하세요",
    data={"name": "dlef", "gender": "male", "dfads": "df"}
)
```


```python
## 클래스 정의 파일(user_profile.py)
from pydantic import BaseModel
from typing import Dict, Any

class ProfileDTO(BaseModel):
  userId: str
  introduction: str
  data: Dict[str, Any]


## 임프트 (main.py)
from test.user_profile import ProfileDTO 

datas = {
			"name": "John Doe", 
			"email": "7Bd0F@example.com", 
			"age": 30, 
			"city": "New York"}

test = ProfileDTO(userId="John Doe", introduction="7Bd0F@example.com", data=datas)

print(test)

```

### 0.3.  JSON 직렬화/역직렬화 

```python
from pydantic import BaseModel

class Person(BaseModel):
  name: str
  age: int 


p = Person(name="yohan", age=29)
print("일반 print : ", p)
# 일반 print :  name='yohan' age=29
print("json print : ", str(p.model_dump_json()))
# json print :  {"name":"yohan","age":29}
print("dict print : ", p.model_dump())
# dict print :  {'name': 'yohan', 'age': 29}
```



### 0.4.  설정 파일 - Pydantic Settings 
https://docs.pydantic.dev/latest/concepts/pydantic_settings/

> FastAPI와 궁합이 좋아서 `.env`/환경변수 기반 설정을 타입 안정성 있게 관리 가능. 


1. *pydnatic-settings 패키지 설치하기*
	```python
	uv add pydantic-settings
	```
	- 이 패키지는 내부적으로 `python-dotenv`를 사용한다 (따로 설치할 필요 없음) 
	- *내부 패키지 `python-dotenv` 역할* : 환경 변수를 `.env`파일에서 읽어와서 처리
	  

>[!tip] Pydantic의 환경 변수 읽기 우선순위
>1. 명시적 전달된 인자
>2. 환경 변수
>3. `.env` 파일 

#### 0.4.1.  표준 구성 방식 - model_config 사용 
```python
# core/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
  app_name: str
  host: str
  port: int
  
  model_config = SettingsConfigDict(
    env_file=".env", # env_file = Pydantic이 어디서 환경 변수를 로드할지 알려주는 설정
    env_file_encoding="utf-8",
    extra="ignore" # 선언 안 된 변수는 무시 
  )

settings = Settings()
print(settings) 
# 출력값 : app_name='간단한 API' host='localhost' port=8081
# .env에서 읽어온 것을 알 수 있다.
```

`model_config`
- Pydantic 모델의 **동작 방식을 커스터마이즈하는 설정을 담는 공간**
- 즉, 이 모델이 **어떻게 작동할지 정의하는 메타데이터**
- pydantic v2에서 나옴
- 사용 예시 
	```python
	from pydantic_settings import BaseSettings, SettingsConfigDict
	
	class Settings(BaseSettings):
	    app_name: str
	    environment: str
	    app_version: str
	
	    # 여기가 model_config
	    model_config = SettingsConfigDict(
	        env_file=".env",              # .env 파일 자동 로드
	        env_file_encoding="utf-8",    # 파일 인코딩
	        env_prefix="APP_",            # 환경변수 접두사(APP_APP_NAME)
	        extra="ignore"                # 선언 안 된 변수는 무시
	    )
	```

`main.py`
```python
import uvicorn
from core.config import settings

def main():
    uvicorn.run(app="app:app", host=settings.host, port=settings.port, reload=True)
```

