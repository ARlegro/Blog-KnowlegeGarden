---
{"dg-publish":true,"permalink":"/DevStudy/Backend/NestJS/NestJS - 유저 생성시 경쟁조건 발생/","noteIcon":"","created":"2025-12-03T14:52:49.538+09:00","updated":"2025-12-13T10:30:08.299+09:00"}
---


## 1.  기존 코드 분석


*✅스키마*
```JS
@Schema()
export class User {
  @Prop({ required: true })
  username: string;

  @Prop({ required: true })
  password: string;
}
```

*✅Register 로직*
```JS
@Injectable()
export class UserService {

  async registerUser(registerUserDto: RegisterUserDto): Promise<RegisterUserResponseDto> {
    await this.validateDuplicatedUsername(registerUserDto.username);
    const hashedPassword = await this.passwordService.hashPassword(
      registerUserDto.password,
    );
  
    const createdUser = await this.userModel.create({
      username: registerUserDto.username,
      password: hashedPassword,
    });

    return RegisterUserResponseDto.fromUser(createdUser);
  }
```
로직 흐름
- username 중복 확인
- password hashing하기
- UserDocument 생성 

### 1.1.  발생할만한 문제 
> 경쟁 조건(race condition)이 존재

사용자명 중복을 검사하고 사용자를 생성하는 사이에 경쟁 조건이 발생할 수 있다. 
두 개의 동시 요청이 모두 중복 검사를 통과한 후 동일한 사용자명으로 생성을 시도할 수 있다.

*✅해결 방법(2가지 절차 갖기)*
1. *데이터베이스 레벨의 unique 제약조건 추가* (DB관련 권장)
	- User 스키마의 username 필드에 **`unique` 인덱스를 설정하여 데이터베이스가 중복을 방지**
	  
2. *MongoServerError 처리 추가*
	- `unique` 제약조건 위반 시 발생하는 에러(코드 11000)를 catch하여 적절한 `ConflictException`으로 변환


>[!tip] MongoDB서버의 중복 키 에러 코드 = 11000
>```JS
>{
  "name": "MongoServerError",
  "code": 11000,
  "keyPattern": { "username": 1 },
  "keyValue": { "username": "yohan" },
  "message": "E11000 duplicate key error collection: app.users index: username_1 dup key: { username: \"yohan\" }"
}



