---
{"dg-publish":true,"permalink":"/DevStudy/Backend/NestJS/NestJs - MongoDB crud/","noteIcon":"","created":"2025-12-03T14:52:49.506+09:00","updated":"2025-12-13T10:30:29.386+09:00"}
---



## 1.  Create 

> 2가지 방식이 있다.
> 1. Model.create() 사용 방식
> 2. Model 반환 후 save() 사용 

```js
	// Model.create() 방식 - 단순 생성
  async createPost(createPostDto: CreatePostDto): Promise<Post> {
    return await this.postModel.create(createPostDto);
  }
  
	// 저장 전 조작이 필요할 때
	const post = new this.postModel({
      title: createPostDto.title,
      content: createPostDto.content,
	});
	// 예: doc.generateSlug(), doc.applyPolicy(userRole) 등
	await doc.save();
```


### 1.1.  검증 - 변환 설정해주는 패턴 


✅검증도 해주는 방법 
```c

```



## 2.  find 


Mongoose에는 여러 조회 메서드를 제공한다
1. `find()` : 여러개 
2. `findOne()` : 1개 
3. `findById(id)` : _id로 한개
4. `.lean()` : 문서 대신 순수 JS객체(POJO)로 받아서 가볍고 직렬화에 좋음 


> [!warning] find는 쿼리 객체를 생성하는 것이다
> - 문서(Documnet)를 찾는 것이 아니라 쿼리 객체를 생성하는 것
> - **당장 쿼리를 실행하려면 `exec()`必*＊ ❗


형태 
```c
// return : Query 객체 - 문서 배열 - Promise<HydratedDocument<Board>[]>
// lean() 사용 시 - Promise<Board[]>
this.boardModel.find(filter, projection, options)
```
- `return` : Query 객체 - 문서 배열 
- `filter` - 필터 조건 
- `projection` - 필드 선택
	- 1은 포함 0은 미포함
	- `{ title: 1, status: 1 }` - title, status 만 포함해서 달라는 뜻
	- `{ description: 0 }` = 제외한다는 뜻 

```js
await this.boardModel
  .find({ status: 'PUBLIC' }, { title: 1, status: 1 })
  .sort({ createdAt: -1 })
  .limit(10)
  .lean();
```

정렬
1. 정렬 

페이징

### 2.1.  Nest JS의 Promise 자동 처리 
```js
@Controller('board')
export class BoardController {

  @Get()
  async getAllBoards(): Promise<Board[]> {
    return this.boardService.getAllBoards();
  }

@Injectable()
export class BoardService {

  async getAllBoards(): Promise<Board[]> {
    return this.boardModel.find().lean().exec();
  }
```
- 위처럼 Promise 타입을 반환하고 await을 사용하지 않아도 문제가 발생하지는 않는다.
- 왜냐하면 NestJS는 라우터 핸들러가 Promise를 반환하면 **해당 Promise가 해결될 때까지 기다렸다고 클라이언트에 응답을 보냄** 


> [!WARNING] 그래도 await를 이용하는 것이 안전하고 명확 


### 2.2.  find + update를 한번에 
> `_id`로 문서를 하나 찾은 뒤 **원자적(atomic)** 으로 업데이트

MongoDB는 find와 함께  update를 할 수 있다.

```js
db.collection.findByIdAndUpdate( filter, update, options )
// findOneAndUpdate({ _id: id }, update, options)의 단축 형태
```
`return` 
- 기본 : 변경 전(before) 문서를 반환
- 옵션 수정 시 업데이트 후 결과를 받을 수 있다.

3대 옵션 - 다 외우기 힘드니 3개 옵션만 외우자
1. `new: true` 또는 `returnDocument: 'after'` : 업데이트 후 document를 돌려 받는다.(권장 : `returnDocument`)
2. `runValidators: true`
	- 업데이트 시 스키마 검증 수행 (기본은 false)
3. `upsert: true`
	- id에 해당하는 documnet가 없으면 **만들어서 반환** 


*⭐안전 패턴*
```js
async updatePost(id: string, dto: UpdatePostDto) {
  const updated = await this.postModel.findByIdAndUpdate(
    id,
    { $set: dto }, // 명시적 $set
    { returnDocument: 'after', runValidators: true },
  ).exec();

  if (!updated) {
    throw new NotFoundException('게시글을 찾을 수 없습니다.');
  }
  return updated;
```

>[!tip] $set 없이 dto만 넘겨도 $set과 동일한 효과
>dto에 존재하는 필드만 바뀌고 없는 필드는 유지 




❓만약 filter 조건이 많다면? - `findOneAndUpdate` 사용 
```js 
async updatePost(id: string, userId: string, dto: UpdatePostDto) {
  const updated = await this.postModel.findOneAndUpdate(
    { _id: id, author: userId },        // 필터로 권한 검증
    { $set: dto },
    { returnDocument: 'after', runValidators: true },
  ).exec();

  if (!updated) {
    throw new ForbiddenException('본인이 작성한 게시글만 수정할 수 있습니다.');
  }
  return updated;
}
```


>[!danger] 단점 
>`save()`와 달리 Mongoose의 훅이 적용되지 않는다.

>[!QUESTION] Mongoose의 훅이란?
>- Mongoose는 **특정 동작 전후에 자동 실행되는 함수**를 걸 수 있게 해준다. 
>- 이걸 “미들웨어(middleware)” or “훅(hook)”이라고 부른다.

```js
// 예: 비밀번호 저장 전에 해시 처리
UserSchema.pre('save', async function (next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

```
- 이렇게 하면 .save()가 호출될 때마다,
- 저장 전에 자동으로 비밀번호 해시화가 수행된다.
- 따라서 이러한 경우가 필요하다면 훅에 저장한 함수를 사용하는 것이 좋음 


### 2.3.  번외 : $set 연산 
> 특정 필드의 값을 새 값으로 갱신할 때 사용하는 업데이트 연산자 

MongoDB는 $set 연산을 제공하며 자주 쓰인다.
