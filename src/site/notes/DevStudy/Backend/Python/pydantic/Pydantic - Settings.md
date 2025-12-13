---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Python/pydantic/Pydantic - Settings/","noteIcon":"","created":"2025-12-03T14:52:50.658+09:00","updated":"2025-12-13T10:27:55.172+09:00"}
---



> Pydantic의 setting관련 기능들을 사용하면 환경 변수나 설정값을 손쉽게 관리할 수 있다.


## 1.  BaseSettings 

### 1.1.  개념 
> Pydantic에서 환경변수나 설정값을 손쉽게 관리하기 위한 클래스

`BaseSettings`를 상속한 클래스의 필드는 환경 변수에서 자동으로 값을 읽어온다 ⭐


```python
class Settings(BaseSettings):
    app_name: str = "MyApp"
    db_user: str
    db_password: str

    class Config:
        env_file = ".env"   # 루트의 .env 파일
        env_file_encoding = "utf-8"

settings = Settings() # type: ignore[call-arg]
```


### 1.2.  modle_config 속성 

`model_config = ConfigDict(...)`

#### 1.2.1.  개념 
> modle의 전역 옵션을 정의하는 속성 

Pydantic V2부터는 `model_config`라는 속성을 사용해서 `BaseSettings`의 동작 방식을 세밀하게 제어할 수 있다.(cf. v1에서는 `config` 사용)

`BaseSettings` 구현체에 들어가 보면 아래처럼 주석으로 적혀있다.
```python
class BaseSettings(BaseModel):
    """
    ....
    All the below attributes can be set via `model_config`.⭐


    Args:
	    ~~~ 
	    ~~~
```

#### 1.2.2.  사용법
- 기본(하드코딩) : `model_config = [json 타입]`으로 선언하면 된다.
- 하지만 hard-coding은 언제나 힘든 법.
- **권장(SettingsConfigDict)** : 따라서 이를 안전하고 IDE에 맞춘 **핼퍼 객체인 `SettingsConfigDict`를 사용하는 것을 권장**

*✅단순 Json으로 할 경우*
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    db_user: str
    db_password: str

    model_config = {
        "env_file": ".env",
        "env_prefix": "MYAPP_",
        "extra": "ignore"
    }
```

*✅ SettingsConfigDict 사용할 경우*
```python
from pydantic import ConfigDict
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    db_user: str
    db_password: str
    
    model_config = SettingsConfigDict(
        env_file=".env",
        env_prefix="MYAPP_",
        extra="ignore",
    )
    
    @property  # 메서드를 속성처럼 쓰게해주는 애노테이션
    def db_url(self) -> str:
        return f"postgresql://{self.DB_USER}:{self.DB_PASSWORD}@{self.DB_HOST}:{self.DB_PORT}/{self.DB_NAME}"    
```


#### 1.2.3.  SettingsConfigDict 주요 옵션들 

| 옵션                              | 설명                                                                                       | 기본값    |
| ------------------------------- | ---------------------------------------------------------------------------------------- | ------ |
| **env_file**                    | `.env` 파일 위치 지정                                                                          | None   |
| **env_file_encoding**           | `.env` 파일 인코딩 지정                                                                         | None   |
| **extra**                       | 모델에 정의되지 않은 필드 처리 방식  <br>• `"ignore"`: 무시  <br>• `"allow"`: 허용  <br>• `"forbid"`: 예외 발생 | forbid |
| **validate_default**            | True로 설정 시, 속성을 재할당할 때마다 유효성 검증 실행                                                       | True   |
| **str_strip_whitespace**        | 문자열 필드 값 앞뒤 공백 자동 제거                                                                     | False  |
| **str_to_lower / str_to_upper** | 문자열을 자동으로 소문자/대문자로 변환(필드값)                                                               | False  |
| **env_prefix**                  | 환경변수 앞에 붙는 prefix 지정 (기본은 `""`)                                                          | `''`   |
| **case_sensitive**              | 환경변수 이름을 대소문자 구분할지 여부                                                                    | False  |
