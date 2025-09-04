---
{"dg-publish":true,"permalink":"/DevStudy/z.etc/obsidian-hosting/","noteIcon":"","created":"2025-06-05T08:57:21.205+09:00","updated":"2025-07-10T18:08:11.312+09:00"}
---


옵시디언으로 메모하다가 다른 블로그에 옮길 때, 깨지는게 심하다. 그리고 내가 원하는 테마가 적용안되는 것e이 너무 불편했다.
이러한 문제를 해결하고자 간단하게 나마 옵시디언으로 블로그를 구축하는 방법을 배울 것 

### 전체 구조 
>옵시디언 전용 웹사이트 만들때 필요한 3가지 

1. Markdwon 파일
2. 정적 사이트 생성기 : 여기선 Digital Garden 플러그인 사용할 것
3. 호스팅 ex. netlify, versal, GitHub Pages 

<br>
## 과정 

### 기본 과정 
1. 깃허브 계정과 netlify 연동
2. Digital Garden 플러그인 설치
3. 배포 
	- https://dg-docs.ole.dev/advanced/hosting-alternatives/ 접속 후 
	- Hosting alternatives ➡ Deploy to Netlify

4. 옵션 설정
		![Pasted image 20250506123240.png](/img/user/supporter/image/Pasted%20image%2020250506123240.png)
	- 깃헙에서 여러 레포 사용 중이라면 Fine-garind tockens 새로 발급 받기 
	- settings ➡ deploy setiings ➡ persconal access tockens
		- 특정 repo만 연동되도록 설정
		- permission : content, pr 섹션이 read & write 되도록 설정
5. Obsidian과 Github연동


### Public 노트 추가 

디지털 가든 플러그인에는 두 가지 중요한 기능이 있다:

- `dg-publish`: 게시 여부
- `dg-home`: 홈 페이지 지정 여부

먼저 새 노트를 만들어 간단히 내용을 작성하고, 속성을 추가

- `dg-publish`: 체크박스 형태로 추가
- `dg-home`: 체크박스 형태로 추가

#### 새 노트를 만들고 add 속성 
![Pasted image 20250506125101.png](/img/user/supporter/image/Pasted%20image%2020250506125101.png)
> 명령 팔레트 : Publish Single Note 하면 그 노트 페이지 배포된다.


## 커스터 마이징 하기 

>	plugin 세팅 ➡` `features ➡ global note settings


eleventy.js 한글 호환 안되는 문제 






