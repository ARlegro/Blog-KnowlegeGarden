---
{"dg-publish":true,"permalink":"/DevStudy/Backend/NestJS/NestJs - 회원가입 기능 (feat. JWT)/","noteIcon":"","created":"2025-12-03T14:52:49.497+09:00","updated":"2025-12-13T10:30:16.520+09:00"}
---



해야할 것 목록
- userSchema 정의
- passwordService 추가
	- 암호화
	- 비교 
	  
- authService 추가
	- 로그인 검증 
- RegisterDTO는 패턴 매칭 
- JWT 모듈들 설치 




## 1.  비밀번호 암호화하기 
> user생성 시 비밀번호를 암호화해서 저장하기 

bcryptjs라는 모듈 사용할 것 

### 1.1.  설치 
```
npm i bcrypte
npm install --save-dev @types/bcrypt
```

### 1.2.  해싱 및 compare
> NestJs에서는 비밀번호 관련 service를 따로 두고 User/AuthService에서 주입받아 사용하는 구조가 깔끔 

```js
import bcrypt from 'bcrypt';
  
@Injectable()
export class PasswordService {
  async hashPassword(password: string): Promise<string> {
    return bcrypt.hash(password, 10);
  }

  async comparePassword(plainPassword: string, hashedPwd: string): Promise<boolean> {
    return bcrypt.compare(plainPassword, hashedPwd);
  }
```

## 2.  중복이름 검색 
몽고 DB의 `exists()`를 사용해서 빠르게 중복 검색할 것 

```js
  async register2User(registerUserDto: RegisterUserDto) {
    const isExist = await this.userModel.exists({
      username: registerUserDto.username,
    });
  
    if (isExist) {
      throw new ConflictException('이미 사용 중인 이름입니다.');
    }
  }
```

## 3.  SignIn 하기

크게 2단계다
1. username or email에 해당하는 document 찾기
2. `bcrypt.compre` 메서드 활용

```js
  async signIn(signInDto: SignInDto) {
    const { username, password } = signInDto;

    const user = await this.userModel.findOne({ username: username });
    if (user && (await bcrypt.compare(password, user.password))) {
      return 'Login Success';
    } else {
      throw new UnauthorizedException('Invalid credentials');
    }
```

# JWT 적용하기

## 1.  사전 준비


### 1.1.  필요한 모듈들 설치 (4가지)

```bash
npm i @nestjs/jwt @nestjs/passport passport passport-jwt
```

```JS
@nestjs/jwt  // nestjs에서 jwt를 사용하기 위한 모듈

@nestjs/passport  // nestjs에서 passport를 사용하기 위한 모듈

passport // passport 모듈

passport-jwt
```


### 1.2.  모듈 등록 
등록할 모듈 
1. JWT
2. Passport


#### 1.2.1.  기본 방법(위험한) - 모듈에 import 
> 사용하는 module에 등록 (보통 `auth.module.ts`)

```js
@Module({
  imports: [
    //...
		PassportModule.register({ defaultStrategy: 'jwt' }),
    JwtModule.register({
      secret: 'secret-key',  // 실전에선 이렇게 ㄴㄴ 
      signOptions: { expiresIn: 60 * 60 }, // 토큰 만료기간(1시간)
    }),
  ],

export class AuthModule {}
```


dlf

### 1.3.  로그인 성공 시 JWT 토큰 생성 
위의 모듈들을 등록했다면 JwtService를 사용할 수 있다.
```js
private readonly jwtService: JwtService,
```

>[!QUESTION] 토큰 생성은 언제❓
>로그인 성공 시 

```js
  async signIn(signInDto: SignInDto): Promise<{ accessToken: string }> {
    const { username, password } = signInDto;
    // ...
  
    if (user && (await bcrypt.compare(password, user.password))) {
      // 유저 토큰 생성 (Secret + Payload 필)
      // 모듈 등록시 설정한 Secret과 payload를 이용한 JWT 토큰 생성
      const payload = { username: user.username };
      const accessToken = this.jwtService.sign(payload);
      return { accessToken: accessToken };
```
*순서*
- payload 생성
- jwtService의 sign 메서드 사용 

![Pasted image 20251022154044.png](/img/user/supporter/image/Pasted%20image%2020251022154044.png)



## 2.  Passport 사용 
