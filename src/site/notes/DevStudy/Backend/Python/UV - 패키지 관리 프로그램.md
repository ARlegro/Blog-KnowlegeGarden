---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Python/UV - 패키지 관리 프로그램/","noteIcon":"","created":"2025-12-03T14:52:50.344+09:00","updated":"2025-12-13T10:27:44.536+09:00"}
---



> [!danger] 기존 Python의 문제 
> 패키지/버전/가상환경/ 빌드 도구 관리 등의 파편화로 개발 곤란

이러한 문제를 발생하고자 2024년 초에 나온게 Rust기반 "**uv**"이다

## 1.  uv란?

### 1.1.  개념 
> Python 프로젝트와 패키지를 쉽고 빠르게 관리해주는 툴 
- 가상환경 생성, 의존성 관리, 파이썬 버전 관리, 패키징, 포매터 등 **서드파티 도구 실행을 한번에 처리 가능**


---
### 1.2.  uv 특징
1. *속도*
	- 설치가 되게 빠르다 
2. *사용 편의성* ⭐
	- `.venv`폴더나 파이썬 버전 등을 신경쓰지 않아도 **실행될 때 관련된 가상환경을 자동 생성,관리**해준다.
	  
3. *기능 통합* 
	- pyenv, poetry, pip, venv 등을 통합 
	- 패키지 설치 시 `uv pip install` or `uv add` or `uv remove`
	- 가상 환경 관리 시 `uv venv`
	- 파이썬 설치 시 `uv pthon install`
	  
4. *스크립트 실행 용이*
	- 가상 환경을 활성화하지 않아도 `uv run`만으로도 python 스크립트 실행 가능 

## 2.  설치
https://docs.astral.sh/uv/getting-started/installation/#standalone-installer
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

## 3.  uv 사용 방법

1. *프로젝트 생성* - `uv init [프로젝트명]`
	```bash
	# 생성 
	uv init uv-demo
	# 포로젝트로 
	cd uv-demo
	```
	- 이렇게 하면 `gitignore` , 파이썬 버전, main.py, README, pyproject.toml이 생성됨(.venv는 실행 시 자동으로 생성됨)<br>![Pasted image 20250910112907.png](/img/user/supporter/image/Pasted%20image%2020250910112907.png)
	- **생성된 파일 중 `pyproject.toml`이 굉장히 중요 ⭐⭐⭐**
		- 역할 : 패키지에 대한 의존성 관리를 할 수 있다. (`package.json`같은 것)
		- 빌드에 필요한 도구나 각종 툴 설정, 프로젝트 메타데이터 등을 한 곳에서 처리 가능 

2. *main 실행 - `uv run main.py`*
	- 개꿀인 점 : run하면 알아서 venv 환경 생성 + 가상환경 activate 함 

3. *패키지 설치 - `uv add [패키지명]`*

 


