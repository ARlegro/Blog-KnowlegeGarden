---
{"dg-publish":true,"permalink":"/DevStudy/Backend/NestJS/TypeORM - 설정/","noteIcon":"","created":"2025-12-03T14:52:49.545+09:00","updated":"2025-12-13T10:30:44.827+09:00"}
---




```bash
npm i @nestjs/typeorm typeorm pg
```
- 총 3개의 모듈을 설치해야 한다.


타입 스크립트 확인 -  `tsconfig.json`
```json
"emitDecoratorMetadata": true,  
"experimentalDecorators": true,
```

## 1.  파일 설정하기 

### 1.1.  config 

```bash
npm i --save @nestjs/config
```


### 1.2.  DataSource 설정(nest js ❌)

> 전역 DB 설정(커넥션)을 하기 위한 설정을 미리 정하는 것 (`TypeOrmMoudle.forRoot(typeORMConfig)`)


```ts
export const AppDataSource = new DataSource({
  type: 'postgres',               // DB 종류 
  host: 'localhost',              // DB 호스트
  port: 5432,                     // DB 포트
  username: 'postgres',           // DB 사용자 이름
  password: 'password',           // DB 비밀번호
  database: 'test_db',            // DB 이름
  entities: [__dirname + '/../**/*.entity.{js,ts}'],  // 사용할 엔티티들
  synchronize: true,              // true면 자동으로 스키마 생성 (개발용)
  logging: true,                  // 쿼리 로그 표시
});
```


> [!WARNING] NestJS에서는 내부적으로 DataSource를 관리함 
>  `TypeOrmModule.forRoot()`가 내부적으로 `DataSource`를 초기화하고 자동 주입함 


### 1.3.  nestjs, typeORMConfig 설정

```ts
export const getTypeOrmConfig = (
  configService: ConfigService,
): TypeOrmModuleOptions => ({
  type: 'postgres',
  host: configService.get<string>('DB_HOST'),
  port: configService.get<number>('DB_PORT'),
  username: configService.get<string>('DB_USERNAME'),
  password: configService.get<string>('DB_PASSWORD'),
  database: configService.get<string>('DB_DATABASE'),
  entities: [__dirname + '/../**/*.entity.{js,ts}'],
  synchronize: true,
  autoLoadEntities: true, // Nest의 엔티티 자동 로드 기능
});
```


### 1.4.  app module에 import

```ts
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }), // 전역으로
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule], // inject할 서비스들이 어디서 오는지 명시
      inject: [ConfigService], // useFactory 함수에 주입할 서비스들
      useFactory: getTypeOrmConfig, // userFactory = 설정을 만드는 함수
    }),
export class AppModule {}
```
설정 
- `imports` : 이 설정이 사용할 모듈 등록
- `inject` : `useFactory`안에 주입할 의존성 목록
- `useFactory` : 주입받은 값으로 **설정 객체를 생성**하는 함수


*실제 동작 순서* 
1. Nest가 ConfigModule을 불러옴
2. `.env` 파일의 값들을 ConfigService에 로드 
3. `useFactory` 함수 실행
	- 실행 시 ConfigService인스턴스를 자동 주입됨 
	- 반환 : DB 설정 객체(`TypeOrmModuleOptions`)
	  
4. Nest가 반환된 DB 설정 객체로 `TypeORM`커넥션 생성 


### 1.5.  classSerializerInterceptor 등록 

```js
  app.useGlobalInterceptors(new ClassSerializerInterceptor(app.get(Reflector), {
    excludeExtraneousValues: false, // @Expose 없어도 전부 직렬화
    enableImplicitConversion: true, // 타입 자동 변환 허용
  }));
```