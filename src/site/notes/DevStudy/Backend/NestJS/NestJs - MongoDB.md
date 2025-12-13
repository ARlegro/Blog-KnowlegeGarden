---
{"dg-publish":true,"permalink":"/DevStudy/Backend/NestJS/NestJs - MongoDB/","noteIcon":"","created":"2025-12-03T14:52:49.488+09:00","updated":"2025-12-13T10:30:24.210+09:00"}
---



https://docs.nestjs.com/techniques/mongodb

## 1.  패키지 설치 
> 설치하면 `MongooseModule`을 root `AppMoudle`에 `import` 가능

```BASH
npm install @nestjs/mongoose mongoose
```
- mongoose : MongoDB를 위한 ODM 라이브러리


## 2.  root 모듈(AppModule)에 import 
> `forRoot()` 메서드 활용 : `mongoose.connect`와 동일한 설정 객체를 인자로 받음

```js 
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    ...
    MongooseModule.forRoot('mongodb://admin:admin123@localhost:27017/board?authSource=admin'),
  ],
})
export class AppModule {}
```


## 3.  스키마 정의 
@Schema, @Prop, @SchemaFactory를 사용 




### 3.1.   Mongoose Model 추가하기
> MongooseModule.forFeature() 메서드를 활용하여 Model 등록

스키마 정의 후 module에 import 추가 
```js
// users.module.ts
import { UserSchema } from './user.document.js';
  
@Module({
  // DI로 모델을 공유 가능
  imports: [MongooseModule.forFeature([{ name: 'User', schema: UserSchema }])],
```
- 기능 : 이 모듈 안에서 **User라는 이름의 Mongoose 모델을 의존성 주입 가능하도록 등록**하는 것 
	- Service or Controller는 아래와 같이 `@InjectModel`을 통해 의존성 주입해서 사용 
		```js
		@Injectable()
		export class UsersService {
			  constructor(@InjectModel(User.name) private readonly userModel: Model<UserDocument>) {}
		```

### 3.2.  다른 레이어에서 활용 

ORM방식에서는 `new Product`같은 방식으로 객체를 생성했다.
이제는 다른 방식 사용할 것 

1. *정의*
	- `HydratedDocumen`와 `SchemaFactory`사용 
		```JS
		export type UserDocument = HydratedDocument<User>;
		
		@Schema()
		export class User {
		  @Prop()
		  name: string;
		}

		export const UserSchema = SchemaFactory.createForClass(User);
		```
	- `HydratedDocumen` - **Mongoose 모델 인스턴스**를 타입 안전하게 표현하기 위한 **TypeScript 타입 도우미**
		- 단순 User 타입만이 아니라 Document 타입도 적용된다.
		- 이로 인해, Mongoose Document 기능을 사용 가능
		- 
	- 

2. *생성자 주입* 
	- `@InjectModel` 사용 - 모델의 이름을 인자로 넘겨주면 됨 
		```c
		@Injectable()
		export class AuthService {
		
		  // X : Model<User>
		  constructor(
		    @InjectModel(User.name) private readonly userModel: Model<UserDocument>, 
		    // 주입받는 모델의 제네릭 타입으로 지정
		  ) {}
		```
	- 인자로 넘겨줄 Model 이름은 module imports부분 forFeature에 정의한 name 속성에 맞아야 됨
	- 타입 : Model<> 


### 3.3.  번외 : document와 schema 차이 
Mongoose와 관련된 class를 정의할 때, 아래와 같이 document와 schema 둘다 export한다. 이 차이는 뭘까❓
```js
//✅document
export type UserDocument = HydratedDocument<User>;

@Schema()
export class User {
  @Prop()
  name: string;
}

//✅스키마
export const UserSchema = SchemaFactory.createForClass(User);
```

1. *UserSchema - 모듈 등록용* 
	- 런타임에 쓰는 Mongoose 스키마 객체 (모듈 등록용)
	- Mongoose가 Model을 생성할 때 참조한다.
	  
2. *UserDocument - 컴파일 전용*
	- 타입스크립트에서 쓰는 Model 인스턴스 타입 

#### 3.3.1.  UserSchema 사용되는 곳 
```js
@Module({
  // 런타임 스키마 등록 - DI로 모델을 공유 가능
  imports: [MongooseModule.forFeature([{ name: 'User', schema: UserSchema }])],
  ...
})
export class UserModule {}  
```
- 런타임에 "User"라는 이름을가진 Schema를 등록한다.

#### 3.3.2.  UserDocument 사용되는 곳 

```js
@Injectable()

export class AuthService {

  // X : Model<User>
  constructor(
    @InjectModel(User.name) private readonly userModel: Model<UserDocument>, 
    // 주입받는 모델의 제네릭 타입으로 지정
  ) {}
```


## 4.  @Schema 

### 4.1.  개념 
> NestJs에서 **Mongoose 스키마를 클래스 기반으로 선언**하기 위한 데코레이터

#### 4.1.1.  💢@Schema가 없다면💢
순수 Mongoose로는 아래처럼 스키마를 정의해야 한다(`new mongoose.Schema`)
```js
const PostSchema = new mongoose.Schema({
  title: String,
  content: String,
  author: String,
});
```

아쉬운점 💢
- NestJs도 OOP스타일을 원하는데 위의 코드는 그렇지 못함
- 그래서 나온 것이 `@Schema`와 `@Prop()` 데코레이터

#### 4.1.2.  @Schema방식 - NestJs
`@Schema`를 통해
1. **클래스 메타데이터**로 Mongoose **스키마 정의 정보 저장** - `@Schema()`
2. **각 필드의 속성을 저장** - `@Prop()`
3. 실제 `mongoose.Schema` **객체로 변환** - `SchemaFactory.createForClass(Post)`
4. NestJS의 `MongooseModule.forFeature()`에서 모델 등록 시 이 스키마를 사용


### 4.2.  옵션들 
|옵션명|설명|
|:--|:--|
|**timestamps: true**|`createdAt`, `updatedAt` 자동 생성|
|**collection: 'posts'**|스키마의 컬렉션 이름 지정|
|**versionKey: false**|`__v` 버전 키 제거|
|**_id: false**|`_id` 생성 방지 (서브도큐먼트용)|
```ts
@Schema({
  timestamps: true,
  versionKey: false,
  collection: 'posts',
})
```



### 4.3.  timestamp: true
스키마에 `@Schema({ timestamps: true })`가 붙어 있어서 Mongoose가 자동으로 createdAt, updatedAt을 추가됨 

```js
@Schema({ timestamps: true }) // createdAt, updatedAt 자동 
export class Post {
```

### 4.4.  lowercase: true
> 문자열을 DB에 저장하기 전에 자동으로 소문자로 변환해주는 옵션

>[!danger] 단, 조회시에서 소문자로 조회해야 한다
>`toLowerCase()`

### 4.5.  예시 - UserSchema
여러 조건이 있는데 그걸 충족시키는 속성을 써볼 것
- 필수 
- unique
- 빈공간 무시하기 
- timestamp사용하기
- version 필요없기 

```js
@Schema({ timestamps: true, versionKey: false })
export class User {

  @Prop({ required: true, unique: true, trim: true })
  username: string;
  
  @Prop({ required: true })
  password: string;
}
```
- 필수 - `required: true`
- unique 제약조건(인덱스도 만들음) - `unique: true`
- 문자열의 앞뒤 공백을 자동으로 제거 - `trim: true`
- createAt, updateAt 자동 생성 추가 - `timestamps: true`
- versionKey 자동생성 제거 : `versionKey: false` 


참고 : DTO에서 미리 trim, lowercase()로 변환도 권장
```js
export class RegisterUserDto {
  @Transform(({ value }) => String(value).trim().toLowerCase())
  username: string;
```


## 5.  스키마 과정 

1. 클래스 메타데이터로 Mongoose 스키마 정의 정보 저장 - `@Schema()`
2. 각 필드의 속성을 저장 - `@Prop()`
3. 실제 `mongoose.Schema` 객체로 변환 - `SchemaFactory.createForClass(Post)`
4. NestJS의 `MongooseModule.forFeature()`에서 모델 등록 시 이 스키마를 사용


