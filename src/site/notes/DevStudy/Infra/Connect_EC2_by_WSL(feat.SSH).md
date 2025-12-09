---
{"dg-publish":true,"permalink":"/DevStudy/Infra/Connect_EC2_by_WSL(feat.SSH)/","noteIcon":"","created":"2025-12-03T14:52:51.727+09:00","updated":"2025-12-09T17:19:47.828+09:00"}
---



✔막혔던 부분 체크 
- 컨테이너끼리 포트 공유 시 
- 주소 접근할때 서비스 명으로 

`MONGO_URI=mongodb://jungle:pwd1234//mongo:27017/dbjungle`

### 0.1.  참고 : 사용된 docker 파일들 (설명 전 배경)

#### 0.1.1.  Dockerfile
```Dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY . /app
RUN pip install --no-cache-dir flask pymongo bs4 requests
#CMD ["python", "app.py"]
CMD ["sh", "-c", "python init_db.py && python app.py"]
```

#### 0.1.2.  docker-compose.yml
```yaml
services:
  my-server:
    build: .
    volumes:
      - movie-data:/app
    environment:
      - MONGO_URI=mongodb://jungle:pwd1234//mongo:27017/dbjungle
    ports:
      - "5000:5000"
    depends_on:
      my-db:
        condition: service_healthy

  my-db:
    image: mongo
    ports:
      - 27018:27017
    environment:
      MONGO_INITDB_ROOT_USERNAME: jungle
      MONGO_INITDB_ROOT_PASSWORD: pwd1234
    volumes:
      - mongo-data:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 5s
      retries: 10
      start_period: 5s
      
volumes:
  mongo-data:
  movie-data:
```




### 0.2.  복합적 문제 (포트 + 서버명)

> 참고 : - `mongodb://<user>:<password>@<host>:<port>/<database>?authSource=admin`

❌잘못된
```python
client = MongoClient("mongodb://jungle:pwd1234@mongo:27018/dbjungle")
```
- 이 코드는 2가지 문제가 동시에 존재한다.

**1‍⃣ 잘못된 호스트 이름 사용** : @mongo
- docker 컨테이너는 서로를 ⭐ **"서비스 이름"으로 DNS 조회**한다  ⭐
- docker-compose 에서 정의된 MongoDB 서비스 이름은 `my-db`이다
- 현재 잘못된 코드에서의 host명은 mongo로 적었다. 이는 존재하지 않는 컨테이너 이름이라 DNS 해석이 되지 않는 문제가 발생 

**2‍⃣ 잘못된 포트 사용** : 27018
- 처음 생각 
	- MongoDB 컨테이너와 application 컨테이너가 다르니까 27018:27017로 설정되어잇는 매핑에 따라 application이 27018로 포트를 접속해야하는 줄 알았다.
- **실제** ⭐
	- 다른 컨테이너라도 **'같은 네트워크'안에 있으면 내부 포트로 접근**해야 한다 
	- 즉, 27018:27017의 27018은 도커 외부에서만 통하는 포트이다.


💚제대로 
```python
client = MongoClient("mongodb://jungle:pwd1234@my-mongo:27017/dbjungle")
```
- 도커 컨테이너끼리 서로 조회하는 것이므로 host명을 서비스명으로 
- 포트도 내부 포트로 접근하도록 


### 0.3.  docker 파일 내부 명령어 설정 (mongodb 헬스체크 + 순차 실행)
> 검색하면 바로 stackoverflow에 나오는 부분이라 가볍게 패스

#### 0.3.1.  몽고DB 헬스체크 
```yaml
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      
      interval: 5s
      retries: 10
      start_period: 5s
```

#### 0.3.2.  순차 실행
 > 일단 특정 파이썬 파일을 실행시키고 그 다음 메인 파일을 실행시켜야 하는 상황 

 원래, 단순 하나 였다면 
```yaml
CMD ["python", "app.py"]
```

2개 순차적 실행
```yaml
CMD ["sh", "-c", "python init_db.py && python app.py]
```


### 0.4.  인증관련 문제 

앱 실행 도중 발생한 오류
```
`pymongo.errors.OperationFailure: Authentication failed.`
```
- MongoDB의 사용자 인증에 실패한 것 

❌잘못된
```python
client = MongoClient("mongodb://jungle:pwd1234@my-db:27017/dbjungle")
```

💚제대로 된
```python
client = MongoClient("mongodb://jungle:pwd1234@my-db:27017/dbjungle?authSource=admin")
```


이유 분석 

> [!INFO] MongoDB URI인증 동작 방식
> 1. URI에 authSource가 있는지 확인 ➡ 있으면 admin 인증 
> 2. authSource가 없다면 ➡ 접속하려는 DB(/jungle)에서 인증 시도 
- 도커 파일을 보면 MONGO_INITDB_ROOT_USERNAME 되어 있다.
- 기본 **`MONGO_INITDB_ROOT_USERNAME` 설정을 통해 생성한 유저는** **"항상 admin DB에서 생성된다"**
	- 따라서, `authSource=admin`은 필수

```python
client = MongoClient("mongodb://jungle:pwd1234@my-db:27017/dbjungle?authSource=admin")
```




>SSH : 다른 컴퓨터에 접속할 때 쓰는 프로그램

>[!SUCCESS]  목표 : WSL 환경에서 AWS EC2인스턴스에 접속하기

>[!tip] EC2 접속 시 SSH를  쓰는 이유 
>- EC2는 서버이기 때문에, 보안상 비밀번호 없이 키파일로만 접속이 가능하다 
>- 배울 것
>	- 키 파일 저장 
>	- 윈도우의 파일 WSL환경으로 복사 
>	- 보안 권한 설정
>	- SSH 방법
>	- 키파일 적용

#### 0.4.1.  키 파일 다운로드
> 다운로드 위치 가정 :  C:\ec2key\jungle.pem 

#### 0.4.2.  WSL 내 .ssh 폴더 생성 
```bash
mkdir -p ~/.ssh
```
- `-p` : 디렉토리가 없으면 생성, 있으면 무시 


#### 0.4.3.  WSL로 키 파일 복사 - cp 
- cp `[윈도우상 위치] ~/.ssh/`
```bash
cp /mnt/c/ec2key/jungle.pem ~/.ssh/
```
- `/mnt/c/...` : Window 디스크를 wsl에서 접근하는 경로 

#### 0.4.4.  키 파일 권한 제한 
- ec2에 보내기 전 키 파일  권한을 제한한다.

> [!INFO] 권한 제한 이유
> - SSH는 보안을 매우 중요하게 여겨서, 키파일이 너무 개방적이면 위험 요소로 간주하고 사용을 거부한다.
> - 따라서 400 모드로 권한을 주면된다.
> - **400 의미** : 나혼자만 읽을 수 있는 상태 (수정도 불가)
```bash
chmod 400 ~/.ssh/jungle.pem
```
<br>

**✔번외 : 권한 제한 확인 방법**
```bash
ls -l ~/.ssh

[정상 예시]
-r-------- 1 root root 1678 May 13 17:36 jungle.pem
```

#### 0.4.5.  EC2에 공개 키 등록 

ssh 연결에 쓰려는 비밀키의 공개키를 복사해서 ubuntu 서버의 ~/.ssh에 파일로 저장 
```bash
echo "id_ed25519.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICubrCDNOTLZE3lglCPWIHjtnT81GSBkeyZtCkJqidGJ icb1696@naver.com" >> ~/.ssh/redis_keys
```
#### 0.4.6.  EC2 접속 
```bash
ssh -i [키페어] ubuntu@ec2pulicIp

ssh -i ~/.ssh/jungle.pem ubuntu@13.125.109.128
```
