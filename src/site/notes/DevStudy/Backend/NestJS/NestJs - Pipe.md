---
{"dg-publish":true,"permalink":"/DevStudy/Backend/NestJS/NestJs - Pipe/","noteIcon":"","created":"2025-12-03T14:52:49.578+09:00","updated":"2025-12-13T10:30:32.562+09:00"}
---



## 1.  Pipe란?
> 입력 데이터의 변환(Transformation)과 유효성 검증(Validation)을 수행하는 클래스 

![Pasted image 20251029140049.png](/img/user/supporter/image/Pasted%20image%2020251029140049.png)

데이터가 컨트롤러에 도착하기 전에 거쳐가는 필터같은 역할 
- Nest는 컨트롤러 메서드가 호출되기 직전에 Pipe를 삽입하고 
- 인수를 수신하고 이에 대해 작동함 

>[!tip] 사전 설치해야 하는 것 
>```bash
$ npm i --save class-validator class-transformer
>```


다양한 옵션들
- https://docs.nestjs.com/techniques/validation

---
## 2.  역할

### 2.1.  역할 1. 변환
- 입력된 데이터를 원하는 타입이나 형태로 바꿔줌
- ex. "123" -> 123 (number)
	  
### 2.2.  역할 2. 검증 - ValidationPipe
- 데이터가 유효하지 않으면 예외 발생시킴
- *대표적인 내장 파이프 - ValidationPipe*
	- DTO클래스에 선언된 유효성 규칙을 자동으로 검증 
	- 원리⭐ : `class-validator`패키지를 이용해 DTO클래스의 데코레이터를 읽고 자동으로 검증을 수행 


#### 2.2.1.  내부 동작 
```js
// 의사 코드 형태 (간단히 표현)
transform(value, metadata) {
  const { metatype } = metadata; // DTO 클래스 정보
  
  // 1️⃣ 변환 로직
  if (this.isTransformEnabled) {
    const object = plainToInstance(metatype, value);
    // plainToInstance는 class-transformer의 함수임
  }

  // 2️⃣ 검증 로직 (class-validator 실행)
  await validate(object);

  return object;
}
```
1. plain 객체를 DTO 인스턴스로 변환
2. DTO의 데코레이터(@IsString, @IsInt 등) 기반으로 검증 수행
3. 검증 통과 시 컨트롤러에 DTO 인스턴스를 전달


#### 2.2.2.  *ValidationPipe 주요 옵션*
- 설정을 통해 세부 동작을 조정할 수 있다.
	```JS
	app.useGlobalPipes(new ValidationPipe({
		transform: true, // DTO 클래스로 자동 타입 변환 
		whitelist: true, // 정의되지 않은 속성 제거 
		forbidNonWhitelisted: true, // 정의되지 않은 속성이 들어오면 예외 발생
	}))
	```

1. `transform: true` ⭐
	- DTO 변환 및 자동 타입 변환 
	- 들어오는 **요청(JSON)을** DTO 클래스의 **타입으로 자동 변환**한다.
		(문자열 "1" ➡ 숫자 1)
	- 기본값 = false : 들어오는 값은 전부 문자열 그대로 유지됨 💢
	  
2. `whitelist: true`
	- 추가 필드를 제거하는 옵션 : 잘못된 필드는 버리고 계속 진행한다.
	- 즉, JSON으로 잘못된 필드를 요청해도 정상 진행 됨 
	- 어떤 속성이 허용되는지 알려줌
	  
3. `forbidNonWhitelisted: true`
	- DTO에 없는 필드가 들어오면 즉시 에러를 발생시킴 💢

>[!QUESTION] whitelist, forbid 옵션을 왜 동시에 쓰는가❓
>1. 만약 whitelist만 true라면❓
>	- DTO 외 필드는 조용히 제거 
>2. 만약 foribid만 true라면❓
>	- DTO외 필드가 뭔지 감지 불가 
>	  
>3. 둘다 true 시 
>	- DTO외 필드 발견 시 오류를 발생 시킬 수 있다.
>	- `whitelist`: DTO 외 필드를 감지
>	- `forbidNonWhitelisted`: 감지된 필드가 있으면 400 오류 발생

---
## 3.  적용 위치 및 적용 
3가지 Level에 적용할 수 있다.

1. Global Level 
2. Controller Level
3. Handle Level

### 3.1.  Global Level
> 앱 전체에 적용. 앱 내 모든 엔트포인트들에 올바르게 변환되고 검증된 데이터가 도달하도록 보장

```js
async function bootstarp() {
		const app = await NestFactory.create(AppModule);
		app.useGlobalPipes(new ValidationPipe());
		await app.listen(process.env.PORT ?? 3000)
}
```


### 3.2.  Controller Level
> 컨트롤러에 적용하여 컨트롤러에 포함된 모든 메서드에 적용

아래와 같은 유효성 검증 데코레이터가 있다고 가정하자
```js 
export class CreateUserDto {
  @IsString()
  @MinLength(2)
  name: string;
```

- 단순히 컨트롤러에서 이 dto를 인자로 받는다고 검증이 적용되지 않는다.\
- `@UsePipes`를 사용해야 가능하다


### 3.3.  Handle Level (메서드 파라미터)
- 특정 파라미터에만 적용



---
## 4.  주의 💢
### 4.1.  주의 1. DTO는 클래스로만 
> - TS는 인터페이스 or 제네릭 타입의 런타임 정보를 저장하지 않는다.
> - 따라서 DTO는 반드시 class로 작성해야 함 

```js
❌
export interface Board {
  id: string;
	 ...

✅
export class Board {
  id: string;
	 ...
```

### 4.2.  주의 2. 일반 import 사용하기

> - import type { }, type-only import 이런거는 runtime에 사라짐
> - 따라서, **DTO를 가져올 때는 일반 import문을 사용**해야 함 

---
## 5.  커스텀 파이프 구현


### 5.1.  기본 
PipeTransform 이라는 인터페이스를 새롭게 만들 Custom Pipe에 구현해줘야 한다.

>[!QUESTION] PipeTransform 인터페이스란❓
>- 모든 Pipe들의 인터페이스 
>- 하나의 메서드만 요구한다.
>```js
>export interface PipeTranfrom<T = any, R= any> {
>	transform(value: T, metadata: ArgumentMetadata): R;
>}
>```



```js
export calss BoardStatusValidationPipe implents PipeTransfrom {
	// 
	transfrom(value: any, metadata: ~ ) {
	}
}
```
이 때, 모든 Pipe는 `transform()`이라는 메서드를 필요로 한다. 

### 5.2.  transform 메서드 
커스텀 파이프를 만들려면 PipeTransform인터페이스를 구현해야 하고 transfrom 메서드를 구현해야 한다.

> - 컨트롤러가 실행되기 직전에 호출되는 함수 
> - 들어온 **요청 데이터를 가공(변환)하거나 검증하는 역할**을 한다 


*Return된 값은 Route 핸들러로 전달됨* 
- 클라이언트 요청 ➡ `transform(value, metadata)` 실행 ➡ `transform()`이 **리턴한 값이 실제 컨트롤러의 파라미터로 전달됨**


2개의 파라미터를 가진다.
1. value : 처리가 된 인자의 값 
2. metadata : 인자에 대한 메타데이터를 포함한 객체


