---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Redis/Setting/Redis Install 및 기본 설정, 실행 명령어/","noteIcon":"","created":"2025-12-03T14:52:50.222+09:00","updated":"2025-12-13T10:24:14.101+09:00"}
---



## 설치 (리눅스 기준)

아래 사이트 참고 
https://redis.io/docs/latest/operate/oss_and_stack/install/archive/install-redis/install-redis-on-linux/

```bash
sudo apt-get install lsb-release curl gpg
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
sudo chmod 644 /usr/share/keyrings/redis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
sudo apt-get update
sudo apt-get install redis
```


---
## Redis 서비스 제어 명령어 (systemd 기반)

Ubuntu 16.04 이상
### ✅ Redis 시작

```bash
sudo systemctl start redis

또는

sudo systemctl start redis-server
```



> 시스템에 따라 서비스명이 `redis` 또는 `redis-server` 둘 중 하나일 수 있으므로 둘 다 시도해볼 수 있습니다.

---

### 🛑 Redis 중지

```bash 
sudo systemctl stop redis
또는

sudo systemctl stop redis-server
```

---

### 🔁 Redis 재시작


```bash
sudo systemctl restart redis

```


---

### 📌 Redis 자동 시작 여부 설정 (부팅 시 자동 실행 여부 설정)

```bash 

sudo systemctl enable redis-server

[자동 시작 해제]
sudo systemctl disable redis-server
```

---

### 🔍 Redis 상태 확인
```bash
sudo systemctl status redis

실행 중인지 확인 (간단 버전)
sudo systemctl is-active redis-server
```
ctl = 씨 티 엘 (영어다) = control

---
