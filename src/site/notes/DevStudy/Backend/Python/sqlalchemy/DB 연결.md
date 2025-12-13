---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Python/sqlalchemy/DB 연결/","noteIcon":"","created":"2025-12-03T14:52:50.326+09:00","updated":"2025-12-13T10:26:35.018+09:00"}
---



*참고: PostgreSQL 기준*

최종 코드 Preview 

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from common.config import DatabaseConfig


dbConfig = DatabaseConfig()

engine = create_engine(dbConfig.sync_db_url())

SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

```PYTHON
class DatabaseConfig(BaseSettings):
    DB_HOST: str = Field(default="localhost")
    DB_PORT: int = Field(default=5432)
    DB_USER: str = Field(default="")
    DB_PASSWORD: str = Field(default="")
    DB_NAME: str = Field(default="")
    DEBUG: bool = Field(default=False)
  
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )
  
    def sync_db_url(self) -> str:
        return f"postgresql://{self.DB_USER}:{self.DB_PASSWORD}@{self.DB_HOST}:{self.DB_PORT}/{self.DB_NAME}"
```


### 0.1.  세팅 

필요 패키지 
```js 
uv add "fastapi[all]" "uvicorn[standard]"
pip install sqlalchemy psycopg2-binary
// pip install sqlalchemy psycopg2
```
- `psycopg2` : 표준 PostgreSQL 드라이버 (운영 서버에서 ㄱ)
	- 운영 서버에서 사용
	- 직접 빌드하고 안정성 확보 
- `psycopg2-binary` : 배포판  PostgreSQL 드라이버 
	- 개발 환경에서 사용 
	- 빌드 과정 없이 바로 쓸 수 있는 binary 배포판 


### 0.2.  DB 연결 - `create_engine()` 사용 

```js
engine = create_engine("postgresql+psycopg2://user:pass@localhost:5432/mydb")

//  ✅ 2. 드라이버 생략 버전 (가능)
engine = create_engine("postgresql://user:pass@localhost:5432/mydb")
```
create_engine 하는 일 
- 실제 DB와 연결을 관리하는 객체를 만듬
- 내부적으로 커넥션 풀(connection pool)을 들고 있음 
- 동기용 엔진을 만든 것 

postgresql + psycopg2 : 어떤 db에 어떤 드라이버로 연결할지 명시 

>[!tip] 정리
>engine = 특정 DB와 접속할 수 있는 커넥션 풀 + 드라이버 정보 + 설정을 함 



### 0.3.  DB 세션을 열기 

> sessionmaker를 통해 DB 연결이 되는 세션 객체 생성 

```python
SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)
def get_db():
    db = SessionLocal() # 새 세션 인스턴스 하나 생성 
    try:
        yield db
    finally:
        db.close() # DB 커넥션 풀 반환, 세션이 들고 있던 객체에 대한 내부 참조 끊어줌 
```

*`sessionmaker` 역할*
- 세션을 만들어주는 공장 함수 
- **이 session이 쿼리도 날리고 트랜잭션도 관리 ⭐**

>[!QUESTION] expire_on_commit=False
>- 기본 값: `True`
>	- commit 이후 세션이 들고 있던 ORM 객체의 값들을 expire 시킴 
>	- 그 후 그 객체 필드 재접근 시, 다시 DB 조회해서 가져옴 
>- `Fasle`
>	- 커밋후에도 메모리에 남은 값을 그대로 씀 
>	- 응답 리턴 시 close와 동기화가 잘 안되는 경우가 많아서 False로 두는 패턴이 많음 

`db.close`
- DB 커넥션 풀 반환
- 세션이 들고 있던 객체에 대한 내부 참조 끊어줌 ⭐
- 더 이상 참조가 없으면 Python GC가 정리함 ⭐


>[!QUESTION] 세션 인스턴스를 생성을 언제하는가?
>요청 하나가 들어올 때마다 호출함 

```python
router = APIRouter()

@router.get("/users")
def list_users(db: Session = Depends(get_db)):
    users = db.query(User).all()
    return users
```
- `/users`라우터로 요청이 들어올 때마다 `get_db()`가 호출된다.
- 이 때, 요청 전용 db 세션이 주입됨 


## 1.  DB 초기화 및 실행

> PostgreSQL + SQLAlchemy 환경에서 초기 세팅을 자동화하는 스크립트
> 1. pgvector 확장 활성화
> 2. SQLAlchemy 모델(base)기준으로 실제 테이블 생성 

이를 통해 DB스키마가 다 만들어지고 pgvector 사용가능한 DB가 됨 

```PYTHON
  

def init_database():
    """데이터베이스 초기화"""
    print("Initializing database...")

    # pgvector 확장 활성화
    with engine.connect() as conn:
        try:
            conn.execute(text("CREATE EXTENSION IF NOT EXISTS vector"))
            conn.commit()
            print("✓ pgvector extension enabled")
        except Exception as e:
            print(f"Error enabling pgvector extension: {e}")
            raise

    # 테이블 생성
    try:
		    # Base.metadata.drop_all(bind=engine)
		    # 이미 존재 시 무시(업데이트 안함) : 없는 테이블만 CREATE 
        Base.metadata.create_all(bind=engine)
        print("Database tables created")

    except Exception as e:
        print(f"Error creating tables: {e}")
        raise
        
    print("Database initialization completed!")

if __name__ == "__main__":
    init_database()
```

### 1.1.  (과정 1) engine.connect()

```python
    with engine.connect() as conn:
        try:
            ....
            print("pgvector extension enabled")
        except Exception as e:
            print(f"Error enabling pgvector extension: {e}")
            raise
```

`with engine.connect() as conn:` : Python의 context manager 전용 문법 
- `with`으로 `connect()`를 감싸면 파이썬이 자동으로 연결을 열고 닫아주는 작업을 함 
- 즉, 아래와 코드가 같음 
	```python
	conn = engine.connect()
	try:
	    # 연결 사용 (SQL 실행 등)
	finally:
	    conn.close()
	```

*✅장점 *
- 자동 자원 정리 : `with`문이 끝나면 자동으로 `close` 호출
- 가독성 향상 

### 1.2.  (과정 2) exectue + commit
```python
from sqlalchemy import text

    with engine.connect() as conn:
        try:
            conn.execute(text("CREATE EXTENSION IF NOT EXISTS vector"))
            conn.commit()
```
- pgvector 확장 설치하는 SQL
- `execute` 문 안에는 명시적인 SQL 객체가 必 ⭐
	- SQLAlchemy 2.0에서부터는 반드시 객체가 必
	- `text()` : SQL 문자열을 안전하게 래핑해주는 객체
	  
- `commit()` 
- 

### 1.3.  (과정 3) Base.metadata.create_all(bind=engine)

```PYTHON
Base.metadata.create_all(bind=engine)
```
- Base: 모든 ORM 모델 클래스가 상속할 공통 부모 클래스
- Base.metadata : 모든 모델 클래스의 테이블 정의를 갖고 있음 
- `create_all(bind=engine)` : 메타데이터를 읽어서 실제 DB 테이블이 없으면 `CREATE` 쿼리를 자동 실행 


참고 - Base 정의
```python
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass
```
- `DeclarativeBase` : SQLAlchemy 2.0에서 새로 도입된 추상 클래스
	- 타입 힌트와 ORM 매핑을 더 명확히 함 
- 상속 예시 
	```PYTHON 
	class User(Base): # Base 상속 
	    __tablename__ = "users"
	    
	    id: Mapped[int] = mapped_column(Integer, primary_key=True)
	    name: Mapped[str] = mapped_column(String(50))	    
	```
	


