---
{"dg-publish":true,"permalink":"/DevStudy/Backend/NestJS/NestJs 시작/","noteIcon":"","created":"2025-12-03T14:52:49.454+09:00","updated":"2025-12-13T10:30:37.884+09:00"}
---


## 1.  설치 

```bash
# 2) npm 글로벌
npm i -g @nestjs/cli
nest new my-app
```

## 2.  의존성 설치 
```bash
npm i @nestjs/common
npm i --save class-validator class-transformer
npm i @nestjs/config js-yaml
```

## 3.  import 제대로 되도록 수정 

```c
// settings.json
  "typescript.preferences.preferTypeOnlyAutoImports": false,
```

```json
  "javascript.suggest.autoImports": true,
  "typescript.suggest.autoImports": true,
  "javascript.preferences.importModuleSpecifier": "relative",
  "typescript.preferences.importModuleSpecifier": "relative",
  "javascript.preferences.importModuleSpecifierEnding": "js",
  "typescript.preferences.importModuleSpecifierEnding": "js",
  "javascript.format.enable": true,
```

## 4.  eslintrc.js 전체 설정 끄기 
NestJS + class-validator를 쓸 땐 이 규칙이 자주 오탐을 내기 때문에  
많은 프로젝트에서 이 규칙을 `off`로 설정
```c
rules: {
  '@typescript-eslint/no-unsafe-call': 'off',
}
```

# 모듈 


## 1.  모듈이란?
> 관련된 기능들을 하나로 묶는 단위 

각 모듈은 도메인에 관련된 Controller, Service, Provider 등을 포함한다.




## 2.  모듈 생성하는 법

nest.js는 명령어로 모듈을 생성할 수 있다.
```bash
nest g module [모듈명]

nest g mo [모듈명]
```
이 명령어를 실행하면
1. `src/[모듈명]/[모듈명].module.ts`파일시 자동으로 생성됨 
2. 1번에서 생성한 모듈을 `src/app.module.ts`에 import함 

뜻 알아보기
- nest : nestcli를 사용하겠다는 소리
- g : genreate 
- module : 




## 3.  모듈 확장 
모듈을 생성한 후 그 모듈에 서비스, 컨트롤러를 추가하는 법 

```bash
nest g service [서비스명]
nest g controller [컨트롤러명] 
```

```bash
nest g controller [컨트롤러명]  --no-spec
```
`--no-spec` : 테스트를 위한 소스 코드를 생성하지 않겠다는 의미 


>[!tip] Nest CLI는 같은 이름의 모듈이 이미 존재하면 자동으로 연결해줌 
>가령 userModule이 있을 때 userService를 생성하면 자동으로 연결하며 생성해줌 



## 4.  typeorm + postgresql
```bash
npm i @nestjs/typeorm typeorm pg
```


## 5.  한 번에 생성하는 법
```js
nest g resource cats
```
- module, controller, service를 한 번에 자동으로 생성 


