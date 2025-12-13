---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Python/fastapi/Fast-API 파일 업로드 방법/","noteIcon":"","created":"2025-12-03T14:52:50.507+09:00","updated":"2025-12-13T10:27:59.988+09:00"}
---



### 어떤 타입을 써야 하는가? 

>[!tip] 권장 타입 - UploadFile 타입

파일 업로드 시 라우터에서 사용할 타입은 2가지가 있다.
1. `bytes`
2. `UploadFile`

```python
@app.post("/files/") 
async def create_file(file: bytes = File()): 
		return {"file_size": len(file)} 

@app.post("/uploadfile/") 
async def create_upload_file(file: UploadFile): 
		return {"filename": file.filename}
```

### 타입 비교 - UploadFile vs Bytes

#### 비교 1. 메모리 효율성

1. `bytes`
	- **파일을 통째로** 메모리에 올림 ➡ 큰 파일이면 메모리 폭발 위험 
	  
2. `UploadFile` ✅
	- 내부적으로 SpooledTemporaryFile 사용하여,
	- **작은 파일을 메모리에 보관**
	- ⭐일정 크기를 넘으면 자동으로 디스크에 spooling(저장) ➡ 대용량 안전한 처리 

#### 비교 2. 추가 메타데이터 제공 

1. `bytes` : 단순 데이터만 있음
2. `UploadFile`
	- `filename`, `content_type` 등 다양한 메타데이터를 제공한다.
	- 실제 객체는 `file` => ex. `file.file`
	- 이로 인해, 이름 및 타입 검증이 가능함 


#### 비교 3. 파일 객체 인터페이스 

1. `bytes` : 단순 바이트열이라 파일 객체 다루듯이(`write()`, `read()` 등) 불가능이다.
2. `UploadFile`
	- 이 객체 자체가 파일처럼 다룰 수 있는 객체이다. ex. `await file.read(), file.seek()`

#### 비교 4. 비동기 지원 

1. `bytes` : 동기적 처리만 가능
2. `UploadFile` : `async` 메서드가 제공되어서 **비동기 처리 가능** 

✅`await`
```python
content = file.read()      # ❌ 동작 안 함 (coroutine object 반환만 함)
content = await file.read() # ✅ 실제로 읽고 결과(content bytes)를 얻음
```


### UploadFile 객체 - Fast API의 마술⭐
![Pasted image 20250911031133.png](/img/user/supporter/image/Pasted%20image%2020250911031133.png)
- 클라이언트가 HTTP 요청을 보낼 때 `multipart/form-data` 형식으로 음성 파일 바이너리를 전송
- FastAPI는 이걸 받아서 임시 저장소에 넣고, `UploadFile` 객체로 감싸서 라우터 함수 인자로 넘김

*읽는 법 - `read()` (binary -> 메모리로)*
1. 전체 한 번에 읽기 - `await audio.read()`
2. 최대 `n` byte만 읽기 - `await audio.read(n)` 

>[!tip] 큰 파일 안전하게 읽는 법 
>청크 단위로 반복해서 읽는게 안전하다 
```python
# 간단 예시 
while True:
	chunk = await audio.read(1024*1024)
	if not chunk:
		break
	tempFile.write(chunk)
``` 



### 임시 파일 저장 - tempfile 모듈 사용
```python
@router.get("/temp")
async def create_temp():
  suffix = ".bin"
  with tempfile.NamedTemporaryFile(delete=False, suffix=suffix, dir="/tmp") as tmp:
 
    tmp_path = tmp.name
  return {"tmp_path": tmp_path}
```
-  `/tmp`폴더에 있음(wsl 기준)
	- `tmp`는 누구나 읽고 쓸 수 있는 임시 저장소 
	- 서버 재부팅 시 날라가는 파일들 
	- 실행 위치랑 전혀 무관
	  
- *delete 속성*
	- True : 자동 삭제
	- False : 수동 삭제 

