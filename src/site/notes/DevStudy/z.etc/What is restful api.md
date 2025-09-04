---
{"dg-publish":true,"permalink":"/DevStudy/z.etc/What is restful api/","noteIcon":"","created":"2025-06-05T08:57:21.136+09:00","updated":"2025-06-05T08:57:21.175+09:00"}
---


> REST = Represtational State Transefer = API 작동 방식에 대한 조건이 있는 S/W 아키텍쳐
### 개념 
> [!SUCCESS] 웹의 모든 데이터(자원)를 URL과 HTTP를 통해 조회하고 조작하는 방식 

- **웹 기반의 자원을 HTTP 프로토콜을 이용해** 접근하는 아키텍쳐 스타일이다.
	- 웹의 모든 자원은 고유한 URL로 식별된다.
- 각 자원은 HTTP메서드(CRUD)를 통해 접근하고 조작된다.
- 두 컴퓨터 시스템이 정보를 안전하게 교환하기 위해 사용하는 인터페이스 
- 클라이언트는 서버의 내부 구조를 몰라도 된다.
	- 정해진 규칙대로 URL, HTTP메서드를 활용하면 된다.
- 쉽게 구현하고 수정할 수 있어 모든 API 시스템을 파악하고 여러 플랫폼에서 사용할 수 있다.
- 유연하고 가벼운 방법을 제공 

### 구성 요소 

1. **고유 리소스 식별자*   
	- 각 자원은 고유한 URI로 식별되어야한다.
	- 자원 : '명사'로 표현
	
2. **메서드 (CRUD)**
	- GET : 리소스에 접근
	- POST : 데이터 전송 
	- PUT : 리소스 전체 수정 
	- PATCH : 리소스 일부 수정 
	- DELETE : 리소스를 제거. 
	  
3. **HTTP 헤더** 
	- HTTP 통신에서 부가 정보를 담는 영역
	- 클라이언트-서버 간에 교환되는 메타데이터 
	- 헤더 종류
		- Content-Type : 요청/응답 본문의 데이터 형식
		- Accept : 클라이언트가 원하는 응답 형식
		- Authorization : 인증 토크 
4. _**데이터**_
	- POST, PUT 및 기타 HTTP 메서드가 성공적으로 작동하기 위한 데이터가 포함될 수 있다.
5. _**파라미터**_
	- 수행해야 할 작업에 대한 자세한 정보를 서버에 제공하는 파라미터가 포함될 수 있다. 
	- 종류 
		- 경로 파라미터 
		- 쿼리 파라미터 
		- 헤더 파라미터
		- 쿠키 파라미터 

### 특징 

**✔ 자원(URI)기반**
- 특정 자원에 접근할 때 URI를 사용한다 
- 이로 인해 접근 방식이 직관적이라는 장점이 있다.

 ✔**상태 없음**
- 상태를 서버에 저장하지 않는 Stateless 구조를 기반으로 한다. 
- **장점** 
	1. 로드 밸런싱 가능 : 서버가 상태를 저장하지 않으므로 요청을 아무 서버에나 분산시킬 수 있다 ➡  Scale-out이 편함 
	2. 서버 리소스 절약 
		- 세션이나 상태 정보를 메모리에 저장할 필요가 없다.
	3.  버그 가능성 줄음
		- 이전 상태를 기억하지 않으므로, 의도치 않은 동작이나 현상을 방지 

✔**HTTP 프로토콜 사용** 
- HTTP의 기본 메서드를 사용한다 (CRUD)
- 웹 서비스에 최적화 되어있다.

 ✔**캐싱** 
- REST API는 캐시 기능의 API응답을 사용하여 캐싱을 제어할 수 있다.
- 캐싱이 가능하다 ➡ 응답 속도가 향상된다.

✔ **표준화** 
✔**다양한 보안 방식 지원** EX. OAuth, JWT 등 


### 장점 
- **직관성**
	- 직관적인 URI구조로 이해하기가 쉽다. 
- **유연성** 
	- 다양한 데이터 포맷을 지원한다  ex. JSON, XML, HTML 등 
	- 클라-서버 분리 : 완전히 독립적으로 동작하는 구조 ➡ API 설계 시 유연하고 변경에 취약하지 않다. 
- **경량성**
	- 무거운 XML을 사용하지 않고 가벼운 JSON 사용 가능
	- 가볍운 이점은 모바일 환경에서도 유리하다 Cuz 모바일은 성능, 속도가 제한적이라 최대한 네트워크 부담을 줄일수록 좋기 때문이다.
- **확장성**
	- 위의 경량성의 장점 때문에, REST는 모바일 환경에서도 좋다.
	- 따라서 웹 ➡ 모바일로의 확장도 용이하다.

### 예시 

이번 예시는 미션때문에 Python이라는 언어를 처음 쓰면서 만들어본 API에 RESTful을 도입해본 코드이다.

| 메서드    | URL                        | 파라미터              | 내용        |
| ------ | -------------------------- | ----------------- | --------- |
| GET    | /                          |                   | 기본        |
| POST   | `/api/movies/<id>/like`    |                   | 좋아요 클릭    |
| GET    | `/api/movies`              | sortMode, trashed | Movie 리스트 |
| PATCH  | `/api/movies/<id>/trash`   |                   | 논리 삭제     |
| PATCH  | `/api/movies/<id>/restore` |                   | 복구        |
| DELETE | `/api/movies/<id>`         |                   | 영구 삭제     |
- URL에 동사형태를 쓰지 않고 명사형태로 넣으려고 노력했다.
- 접근할 자원은 파라미터로 전달하고 
- 추가 데이터가 필요한경우 쿼리파라미터로 전달하였다

```python
[기본]
@app.route("/")
def index():
    return render_template("index.html")

[POST - 좋아요 클릭]
@app.route("/api/movies/<id>/like", methods=["POST"])
def like_movie(id):

[GET - Movie 리스트 : 파라미터값에 따라 미삭제/삭제 된 영화 가져오기]
@app.route("/api/movies", methods=["GET"])
def get_movie_list():
		# 파라미터값 
    sortMode = request.args.get("sortMode", "likes")
    trashed = request.args.get("trashed", "false").lower() == "true"

[PATCH - movie를 논리삭제로 수정]
@app.route("/api/movies/<id>/trash", methods=["PATCH"])
def trash_movie(id):

[PATCH - 논리삭제된 movie를 복구]
@app.route("/api/movies/<id>/restore", methods=["PATCH"])
def retore_movie(id):

[PATCH - movie 영구 삭제]
@app.route("/api/movies/<id>", methods=["DELETE"])

def delete_movie(id):
```


