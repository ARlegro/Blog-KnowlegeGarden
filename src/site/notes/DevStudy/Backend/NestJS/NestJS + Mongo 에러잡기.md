---
{"dg-publish":true,"permalink":"/DevStudy/Backend/NestJS/NestJS + Mongo 에러잡기/","noteIcon":"","created":"2025-12-03T14:52:49.553+09:00","updated":"2025-12-13T10:30:35.289+09:00"}
---



```
npm install --save-dev @types/mongodb
```

# 예외 필터 공부

## 1.  Global Exception Filter 
NestJS는 애플리케이션 전체에서 **처리되지 않은 모든 예외**를 처리하는 내장 **Exception Layer**를 제공

기본적으로 내장된 Global Exception Filter에 의해 수행됨 
- G.E.F는 `HttpException` 및 그 서브클래스 타입(상속한)의 예외를 처리한다
- 기본 JSON 응답을 생성
	```JSON
	{
	  "statusCode": 500,
	  "message": "Internal server error"
	}
	```

## 2.  HttpException 클래스 
`@nestjs/common` 패키지에서 노출되는 내장 **`HttpException`** 클래스를 제공

보통 RESP API 기반 애플리케이션에서는 특정 오류 조건이 발생할 때 HTTP 응답 객체를 전송한다.

기본 하드코딩으로 예외 던져보면 
```js
@Controller('exception')
export class ExceptionController {

  @Get()
  getException() {
    throw new HttpException('exception', 500);
  }
```

아래처럼 응답 옴 
```json
{"statusCode": 500,"message": "exception"}
```

HttpException 클래스느 아래처럼 생김 
```js
constructor(response: string | Record<string, any>, status: number, options?: HttpExceptionOptions | undefined);
```
`HttpException` 생성자는 응답을 결정하는 두 개의 필수 인수를 받는다
1. `response` : JSON 응답 본문을 정의 (문자 or 객체 가능)
2. `status` : HTTP 상태 코드를 정의 
	- `@nestjs/common`에서 가져온 **`HttpStatus`** enum을 사용 (추천) ⭐
		```js
		 getException() {
			    throw new HttpException('exception', HttpStatus.INTERNAL_SERVER_ERROR);
		 }
		```
3. (옵셔널) options : 원인을 제공하는데 사용할 수 있음 


## 3.  Custom Exception 

```js
export class ForbiddenException extends HttpException {
  constructor() {
    super('Forbidden', HttpStatus.FORBIDDEN);
  }
}
```

# 예외 필터 (Exception Filters)
> 예외 Layer에 대한 제어를 원할 경우 사용 (ex. 다른 JSON 반환, 로깅 추가 등)


## 1.  예외 필터 만들기 
예외 필터를 만들기 위해서는 `ExceptionFilter<T>`인터페이스를 구현해야 한다. 이 인터페이스는 `catch`메서드 제공을 요구 

### 1.1.  예외 필터 예시 
```js
import { ArgumentsHost, Catch, ExceptionFilter, HttpException } from '@nestjs/common';

@Catch(HttpException)
export class GlobalExceptionFilter implements ExceptionFilter {
  catch(exception: any, host: ArgumentsHost) {
    throw new Error('Method not implemented.');
  }
}
```
`@Catch(HttpExcetpion)`
- `HttpExcetpion`클래스 인스턴스들의 예외를 포착 

*catch 메서드*
1. `exception` 인수
	- 현재 처리 중인 예외 객체 
	  
2. `ArgumentHost` 인수 - 강력한 유틸리티 객체
	- 원래 요청 핸들러(예외가 발생한 컨트롤러)로 전달되는 **`Request`** 및 **`Response`** 객체에 대한 참조를 얻는다.

### 1.2.  ArgumentHost 심화
> 핸들러의 인수에 대한 추상화

NestJS는 애플리케이션을 쉽게 작성할 수 있도록 돕는 몇 가지 유틸리티 클래스를 제공한다(ex. `ArgumentHost`, `ExceptionContext`)
- 현재 실행 Context에 대한 정보를 제공하고 
- 제네릭 가드, filter, interceptor 구축 가능 

그 중 `ArgumentHost`에 대해 공부할 것 

*❓`ArgumentHost`역할❓*
- 핸들러로 전달되는 인수를 검색하는 메서드를 제공 
- 예를 들어, 인수를 가져올 적절한 Context(ex. HTTP, RPC, 웹소켓)를 선택할 수 있도록 함 

*배열의 캡슐화*
```js
const [req, res, next] = host.getArgs();
```
- **`[request, response, next]`** 배열을 캡슐화한 것이다. 
- request : 요청 객체
- response : 응답 객체
- next : 애플리케이션의 요청-응답 주기를 제어하는 함수 

Context에 따라 다르게 처리하고 싶다면 아래처럼 if문 쓰면 된다
```js
if (host.getType() === 'http') {
  // 일반 HTTP 요청 (REST) 컨텍스트에서만 중요한 작업을 수행합니다.
} else if (host.getType() === 'rpc') {
  // 마이크로서비스 요청 컨텍스트에서만 중요한 작업을 수행합니다.
} else if (host.getType<GqlContextType>() === 'graphql') {
  // GraphQL 요청 컨텍스트에서만 중요한 작업을 수행합니다.
}
```

#### 1.2.1.  switchToHttp()
> - HTTP 애플리케이션 Context에 적합한 HttpArgumentHost 객체를 반환한다.
> - 이렇게 반환된 객체는 원하는 객체를 반환하는 2가지 유용한 메서드가 제공된다
> 	1. getRequest
> 	2. getResponse


```js
const ctx = host.switchToHttp();
const request = ctx.getRequest<Request>();
const response = ctx.getResponse<Response>();
```





## 2.  필터 바인딩 
> UseFilters()를 사용해서 만든 **예외 필터를 컨트롤러에 바인딩** 해줘야 한다.

### 2.1.  Global ⭐추천
> 글로벌 scope 필터 

전체 애플리케이션, 모든 컨트롤러, 모든 경로 핸들러에서 사용된다.

```js 
async function bootstrap() {
  ...
  app.useGlobalFilters(new GlobalExceptionFilter());
```
앱 전체에 공통 적용할 수 있다.
- 모든 컨트롤러
- 모든 서비스
- 모든 라우트 

이렇게 하면 컨트롤러에서 `@UseFilters`사용할 필요가 없다


### 2.2.  컨트롤러에 - UseFilters()
```js
@Controller('exception')
export class ExceptionController {

  @Get()
  @UseFilters(new GlobalExceptionFilter()) // 🥊이건 너무 많은 인스턴스를 쓰는 상황
  getException() {
```
UseFilters를 사용해서 



### 2.3.  전역으로 한 번만 등록해서 바인딩 

- 컨트롤러마다 `GlobalExceptionFilter`인스턴스를 생성하는 방식은 메모리 낭비가 심하다
- 따라서 하나의 인스턴스를 공유하는 방식이 권장됨 
- 이 때 사용하는 것이 `APP_FILTER` 토큰 


*설정 단계*
1. `APP_FILTER` 토큰으로 모듈에 등록 (DI 안정)
2. 컨트롤러에서 바로 사용 

```JS
@Module({
  
	//...
  providers: [
    {
      provide: APP_FILTER,
      useClass: GlobalExceptionFilter,
    },
  ],
})

export class AppModule {}
```

```JS
import { GlobalExceptionFilter } from '../common/global-exception.filter.js';

@Controller('exception')
export class ExceptionController {

  @Get()
  @UseFilters(GlobalExceptionFilter) // 인스턴스 재사용 
```






# 전역 필터 잡기 
NestJS의 전역 필터(Global Exception Filter)는 NestJS에서 발생하는 모든 예외를 한 곳에서 잡아서 공통 처리한다
(스프링의 `@ControllerAdvice` + `@ExceptionHandler` 역할을 합쳐놓은 구조)



참고 - MongoServerError 클래스는 MongoDB Node.js 드라이버에서 제공하는 클래스이기에 아래 패키지 설치 必 (monggose 소속이 아님)
```bash
npm i mongodb
```


전역 등록
```js
// app.module.ts
async function bootstrap() {
	... 
  app.useGlobalFilters(new MongoExceptionFilter());
```


```js
  

@Catch(MongooseError, MongoServerError)
export class MongoExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
  
    if (exception instanceof MongoServerError && exception.code === 11000) {
      const httpError = new ConflictExcepton('이미 사용 중인 값입니다.');
      const status = httpError.getStatus();
      const payload = httpError.getResponse();
      response.status(status).json(payload);
      return;
    }
  
    response.status(HttpStatus.INTERNAL_SERVER_ERROR).json({
      statusCode: HttpStatus.INTERNAL_SERVER_ERROR,
      message: 'Database operation failed.',
    });
  }

}
```