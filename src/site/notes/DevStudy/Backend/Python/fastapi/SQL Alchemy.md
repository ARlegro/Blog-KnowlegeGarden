---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Python/fastapi/SQL Alchemy/","noteIcon":"","created":"2025-12-03T14:52:50.497+09:00","updated":"2025-12-13T10:29:16.786+09:00"}
---



필수 패키지 
```
pip install fastapi uvicorn sqlalchemy
```



그 외 패키지
- alembic : 스키마 버전 관리 
- `psycopg2-binary` : Postgresql
- `sqlalchemy[asyncio]` : 비동기 orm 


>[!tip] PostgreSQL 기준 전체 설치
>```BASH
>pip install fastapi uvicorn sqlalchemy psycopg2-binary python-dotenv alembic
>```


전체 버전 - `uv add -r requirements.txt`
```text
fastapi
uvicorn[standard]
psycopg2-binary
alembic
crawl4ai
asyncio
httpx
pydantic
pydantic-settings
sqlalchemy[asyncio]
asyncpg
python-multipart
# sqlalchemy
```