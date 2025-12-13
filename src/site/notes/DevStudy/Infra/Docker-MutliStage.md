---
{"dg-publish":true,"permalink":"/DevStudy/Infra/Docker-MutliStage/","noteIcon":"","created":"2025-12-03T14:52:51.746+09:00","updated":"2025-12-13T09:26:31.974+09:00"}
---


참고 : https://docs.docker.com/build/building/multi-stage/

> Dockerfiles를 최적화하는데 큰 도움 (읽기도 쉽고)

### 왜 Multi-Stage 빌드가 필요한가?

전통적인 Dockerfile 방식은 빌드 도구(Gradle, Maven 등)와 소스 코드, 심지어 테스트 리소스까지 모두 포함된 **무거운 이미지**를 만들 수밖에 없습니다.
> 이러한 방식은 보안, 용량, 배포 속도 측면에서 매우 비효율적입니다.

이를 해결해주는 방식이 바로 `Multi-Stage build`방식이다.

> [!WARNING] 정리 : 기존 방식(single-stage)의 문제 
> - 빌드 도구, 소스 코드가 전부 포함
> - 용량 증가
> - 보안상 위험
> - 배포/다운로드 시간 증가 


### Multi-Stage의 핵심: 빌드와 실행의 분리

#### Concept
Docker는 최종 이미지를 생성할 때, **마지막 `FROM` 명령으로 시작하는 스테이지**만 가지고 실제 이미지를 만듭니다.
Multi-Stage 방식은 이를 이용하는 것으로 빌드 전용 stage와 실행 전용 Stage를 나누는 방식입니다.
1. Build 전용 stage : 소스코드, 빌드 도구, 테스트 등 포함 
2. 실행 전용 stage : 실제 배포 시 필요한 결과물**만** 포함 

#### Multi-stage 예시 
아래의 예시는 2개의 스테이지를 갖고 있습니다.
```dockerfile
FROM openjdk:21-jdk AS build
WORKDIR /app
COPY . .
RUN chmod +x gradlew && ./gradlew clean build -x test

FROM openjdk:21-jdk
WORKDIR /app
COPY --from=build /app/build/libs/*SNAPSHOT.jar app.jar
CMD ["java","-jar","app.jar"]
```
이 dockerfile 설정은 multiStage방식으로 작성한 것이다.
#### 예시 단계별 상세 분석 <br>
**1‍⃣ 첫 번째 stage : build 전용** 
- `AS build`를 통해 **이 스테이지의 이름을 `build`로 명명**
- `COPY . .`로 전체 소스 복사
- `./gradlew clean build -x test`로 빌드
- 여기서 `RUN chmod +x gradlew && ./gradlew clean build -x test` 명령어를 통해 컴파일된 결과물(artifact)인 **`SNAPSHOT.jar` 파일이 생성**된다.

> [!INFO] 이 단계에서 쓰인 빌드 도구와 소스는 최종 이미지에 포함되지 않을 것이다. 단지, 중간 캐시로 활용될 것 

**2‍⃣ 두 번째 stage : runtime용** 
- 런타임에 필요한 최소한의 환경만 유지한다.
- `COPY --from=build ...`  : build라는 이름을 가진 stage의 결과물(artifact)만 가져옴
- Stage(`build`)에서 빌드된 바이너리(`/app/build/libs/*SNAPSHOT.jar`)를 최종 이미지의 `/app.jar` 디렉토리로 복사
- 최종 이미지에는 빌드 도구(gradle)나 소스 코드 없이 실행에 필요한 바이너리만 포함
- 즉, **최종 배포 이미지에는 JAR만 존재**

> [!WARNING] 이렇게 FROM절이 2개여도 실행 시 script는 1번으로 가능

### Q&A - 헷갈렸던 점 

❓`COPY . .`는 빌드 초기에 소스코드를 다 복사하는데, 이미지가 왜 작아지는가?
- 첫 번째 스테이지는 **최종 이미지에 포함되지 않는다**
- Docker는 최종 이미지를 생성할 때, **마지막 `FROM` 명령으로 시작하는 스테이지**만 가지고 실제 이미지를 만든다.

❓외부 이미지도 COPY 가능?
- 외부 이미지도 현재 스테이지에 복사할 수 있다. 
	```dockerfile
	COPY --from=nginx:latest /etc/nginx/nginx.conf /nginx.conf
	```
- 공식 이미지의 파일도 특정 경로로 복사할 수 **있음**

### 최적화 가능한 점 
물론, 빌드 stage에서도 COPY . . 로 인해 context자체는 커질 수 있다.
이러한 부분은 **`.dockerignore`로** 필요없는 파일들 (ex. `.git, build/)` 을 **제외해야 최적화** 가능

### 결론 
>[!tip] Multi-Stage방식은, 최종적으로 배포되는 이미지에는 오직 실행에 필요한 최소한의 파일만 포함시켜 이미지 크기를 획기적으로 줄일 수 있는 강력한 기능

>개발 편의성과 배포 효율성을 모두 높일 수 있으므로 적극적으로 활용하는 것이 좋다

--- 
