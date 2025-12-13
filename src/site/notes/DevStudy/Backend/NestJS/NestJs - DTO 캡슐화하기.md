---
{"dg-publish":true,"permalink":"/DevStudy/Backend/NestJS/NestJs - DTO 캡슐화하기/","noteIcon":"","created":"2025-12-03T14:52:49.443+09:00","updated":"2025-12-13T10:30:19.720+09:00"}
---



> 핵심 = Object.freeze(this)

## 1.  예시
### 1.1.  전체 코드 - record 스타일
```js
export class UserResponseDto {
  public readonly id: string;
  readonly name: string;

  private constructor(params: { id: string; name: string}) {
    this.id = params.id;
    this.name = params.name;
    Object.freeze(this);
  }
  
	static fromUser(user: UserDocument) {
    return new RegisterUserResponseDto({
      id: user._id.toString(),
      username: user.username,
    });
  }
}
```

### 1.2.  readonly + freeze()

```js 
export class UserResponseDto {
  ...
  public readonly email: string;
  constructor(params: { id: string; name: string; email: string }) {
    ...
    Object.freeze(this);
  }
```

불변성을 위한 안전장치이다.
spring의 record타입과 비슷 

>[!QUESTION] Object freeze()❓
>- 객체를 완전히 읽기 전용으로 만드는 JS 내장 함수
>- 한번 동결하면 객체의 property를 바꾸거나 수정하거나 삭제할 수 없다.



>[!QUESTION] freeze()걸었는데 왜 또 readonly❓
>- freeze()걸면 이미 수정이 불가능하지만(런타임용)
>- **readonly를 걸으면 타입 차원에서 막을 수 있다**


### 1.3.  public 
> nestJS에서는 접근제어자를 명시하지 않으면 항상 `public`이다.

```js
export class RegisterUserResponseDto {
	
  (public) readonly id: string;
  public readonly username: string;
```

>[!QUESTION] 캡슐화를 하는데 왜 private이 아닌 public으로 둘까❓
>- NestJS에서는 직렬화 시 `public`이 아니면 안된다.
>- `class-transformer`가 내부적으로 DTO 직렬화 시 public필드를사용하기 때문
>- 물론 `private` + `@Expose()`로 할 수 있지만 매 필드마다 하는 것은 불편하다.
>- 따라서 public으로 놓고 readonly (`Object.freeze(this)`) 거는 것이 좋다

