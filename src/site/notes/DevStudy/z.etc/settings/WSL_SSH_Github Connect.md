---
{"dg-publish":true,"permalink":"/DevStudy/z.etc/settings/WSL_SSH_Github Connect/","noteIcon":"","created":"2025-12-03T14:52:46.483+09:00","updated":"2025-12-09T17:19:49.080+09:00"}
---



**✅ 확인**
```
ls -al ~/.ssh
```


**✅ 키 생성 - `ssh-keygen -t ed25519 -C [깃허브주소]`**
```
(crafton) icb1696@DESKTOP-EFD5M71:~/jungle$ ssh-keygen -t ed25519 -C "icb1696@naver.com"
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/icb1696/.ssh/id_ed25519): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/icb1696/.ssh/id_ed25519
Your public key has been saved in /home/icb1696/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:/HlFvGvF5hQY7xybjOi9wpfZZby3gJQfEiJPsxMzvtM icb1696@naver.com
The key's randomart image is:
+--[ED25519 256]--+
|             .   |
|             .+  |
|       . B . .o+ |
|       .= B +.=o=|
|        S= = ooB=|
|         .*.=..*+|
|         o+E.+*.+|
|          .+ =oo.|
|            o. .o|
+----[SHA256]-----+
```


**✅ SSH 에이전트 실행

에이전트 실행
```
eval "$(ssh-agent -s)"  
```
- ssh-agent : SSH키를 메모리에 저장하는 프로그램 
- 이 명령어 뜻 : ssh-agent를 백그라운드에서 실행시키고 출력된 환경 변수들을 현재 셸에 적용시킨다.
- eval : 문자열 안에 있는 리눅스 명령어를 한번 더 실행 시킨다.

>[!tip] tip
>- 즉, $(ssh-agent -s) 는 ssh-agent의 실행 결과를 반환하는데(환경변수 관련) eval을 붙이면 이거를 단순히 반환만 하는게 아니라 한번 더 실행하게 된다.
>- 위의 ssh-agent -s의 경우 환경변수들을 출력하는데 이걸 한번 더 실행한다면 실제 환경변수로 등록이 되는 것이다.


**✅ssh-agent에 키 등록** 
```
ssh-add ~/.ssh/id_ed25519
```
- 지정한 개인키(id_ed25519)를 ssh-agent에 등록하는 것 
- ssh-add : 등록한다  ➡ ~~~ id_ed25519 키를 

**✅ Github에 공개 키 등록** 

1. 일단 생성한 ssh키를 확인하기 위해 아래와 같이 명령어 입력
	- `cat ~/.ssh/id_ed25519.pub`
	  
2. 출력된 내용을 전부 복사해서 ssh키로 등록하면 된다.



### 번외 : `ed25519`란?

>[!EXAMPLE] 개념 
>- 공개키 암호화 방식 중 하나
>- GitHub, SSH, GPG 등에서 인증용으로 사용됨
>- 기존 RSA 방식보다 더 빠르고, 더 짧은 키로 더 높은 보안성 제공
>- 요즘은 GitHub에서도 `RSA` 대신 `ed25519`를 권장

정보처리기사 필기([[DevStudy/z.etc/Fundamentals/정보처리기사/정보시스템_구축관리/암호화\|암호화]] ) 에서 나온 ECC(타원 곡선 암호화)의 종류네.
그러다 보니 적은 키로도 더 빠르고 안전한 키 

#### 비교: `ed25519` vs `RSA`

| 항목    | ed25519                   | rsa                  |
| ----- | ------------------------- | -------------------- |
| 보안 수준 | 매우 강력 (모던한 알고리즘)          | 충분하지만 ed25519보단 느림   |
| 키 길이  | 고정 (256bit)               | 가변 (보통 2048~4096bit) |
| 속도    | 빠름                        | 느림                   |
| 지원 여부 | GitHub, SSH 최신 버전에서 기본 지원 | 기본 지원                |
