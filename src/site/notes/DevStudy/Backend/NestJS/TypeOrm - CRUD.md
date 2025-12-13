---
{"dg-publish":true,"permalink":"/DevStudy/Backend/NestJS/TypeOrm - CRUD/","noteIcon":"","created":"2025-12-03T14:52:49.568+09:00","updated":"2025-12-13T10:30:48.557+09:00"}
---


## 1.  CREATE


### 1.1.  관계 매핑은 “객체 참조”를 통해 작동함
```JS
@Entity('post', { schema: 'public' })

export class Post extends BaseTimestampEntity {

  @Column({ type: 'text', name: 'title' })
  title: string;

	... 
  @ManyToOne((): typeof Users => Users, { onDelete: 'RESTRICT' })
  @JoinColumn([{ name: 'writer_id', referencedColumnName: 'id' }])
  writer: Users;
}
```
- `writer: Users` 이런 필드가 있을 때, 보통 JPA에서는 그냥 USER객체 자체를 넣었다.
- 하지만, nestJS는 아래와 같은 방법으로도 매핑이 가능하다.
	```JS 
    this.postRepository.create({
	      ...createPostDto,
	      writer: { id: userId }, // typeorm의 관계 매핑은 객체 참조 
    });
	```

> TypeORM은 내부적으로 **id만 들어 있어도 관계 매핑이 가능**

>[!QUESTION] 이유
>**id만 있는 객체를 넣으면** ➡ **자동으로 FK로 인식**해서 `INSERT` 시 `writer_id` 컬럼에 값 채움 


### 1.2.  date 처리 

> 간단하게 YYYY-MM-DD 형식으로만 쓰고 싶다면 아래처럼 해라
> 1. DTO에서 string으로 받돼 YYYY-MM-DD 형식으로 검증
> 2. entity 컬럼 타입은 string으로 

```js
const YMD = /^\d{4}-\d{2}-\d{2}$/;
export class CreatePostDto {

  @IsOptional()
  @Matches(YMD, { message: '시작일은 YYYY-MM-DD 형식으로 입력해줘' })
  startDate?: Date;
  
  @IsOptional()
  @Matches(YMD, { message: '시작일은 YYYY-MM-DD 형식으로 입력해줘' })
  endDate?: Date;
```


```js
@Column({ type: 'date', name: 'start_date', nullable: true })
start_date: string | null;

@Column({ type: 'date', name: 'end_date', nullable: true })
end_date: string | null;
```

- 장점: 시간대 이슈/하루 밀림이 없다
- DB(Postgres) `DATE` ↔️ 코드 `string('YYYY-MM-DD')` 1:1 매핑.

## 2.  UPDATE

> create와 마찬가지로 엔티티를 `save()`메서드로 활용 

```js


```

>[!tip] 어떻게 구분되는가❓
>- 객체에 id가 있으면 : INSERT처리
>- 객체에 id가 없으면 : UPDATE처리 


### 2.1.  merge 방식


```js
  async update(userId: string, dto: UpdatePostDto) {
    const entity = await this.postRepository.findOne({
      where: { id: dto.id, writer: { id: userId } },
    });
    
    if (!entity) {
      throw new NotFoundException('Post update failed');
    }
  
    const merged = this.postRepository.merge(entity, dto);
    const savedPost = await this.postRepository.save(merged);
    return this.toPostResponseDto(savedPost);
  }
```
- 총 2번의 쿼리 
- **과정**
	1. 조건에 맞는 entity를 찾는다.
	2. merge 처리를 통해 dto에 있는 필드로 덮어쓴다.
	3. 새로운 엔티티 객체를 save한다.
	   
- `merge(entity, dto)`
	- DB에서 조회한 기존 엔티티의 속성들에 DTO에서 전달된 값만 덮어써서 새로운 엔티티 객체를 만든다.
	- 실제 DB는 아직 반영되지 않은 상태 
	


### 2.2.  preload 방식 


> DB에서 기존 엔티티를 찾아서 DTO 데이터로 병합한 뒤 새 엔티티 객체를 반환

**내부 동작 - find + merge**
- `findOneBy({id})`로 해당 엔티니를 DB에서 가져옴
- `Object.assign(entity, dto)` 형태로 필드가 자동 병합됨
- 병합된 새 Entity 인스턴스를 리턴 

```JS
async update(id: string, updatePostDto: UpdatePostDto) {
  // 기존 post를 찾고, DTO의 값으로 덮어쓴 엔티티를 반환
  const post = await this.postRepository.preload({
    id,
    ...updatePostDto,
  });

  // 없는 경우 예외
  if (!post) {
    throw new NotFoundException(`Post with id ${id} not found`);
  }

  // 변경사항 저장
  return await this.postRepository.save(post);
}
```
- id가 필수이다.
  
> [!WARNING]  단점
1. id가 필수이다.
2. 엔티티를 로드해야되어서 느리다. 

### 2.3.  update() 방식 
> 엔티티를 메모리로 로드하지 않고 바로 SQL날리는 방식 


#### 2.3.1.  개념 및 예시

- 단순 필드 수정 시 효율적이다.
- 빠르다.

```js
const result = await this.postRepository.update(foundedPost.id, updatePostDto);

if (result.affected === 0) {
  throw new NotFoundException(`Post with id ${foundedPost.id} not found`);
}
```

> [!WARNING] 단점
> - 엔티티 훅(@BeforeUpdate)이나 validation, transformer를 전혀 안 거친다.


#### 2.3.2.  반환 값 구조 
```js
const result = await this.postRepository.update(foundedPost.id, updatePostDto); console.log(result)
```
`result` 구조
```json
{   
		"generatedMaps": [],   
		"raw": [],   
		"affected": 1  ⭐  
}
```

| 필드              | 설명                                   |
| --------------- | ------------------------------------ |
| `affected`      | 영향을 받은 행(row) 개수 (0이면 실패, 1 이상이면 성공) |
| `raw`           | DB가 반환한 원시 결과 (일반적으로 빈 배열)           |
| `generatedMaps` | `save()`에서만 채워짐 (update()는 거의 항상 빈값) |





## 3.  Find 


### 3.1.  기본 Pagination 

#### 3.1.1.  Repsitory API (find, findAndCount)
`find`, `findAndCount`를 활용해서 Pagination을 구현할 수 있다.

```js
    const result = await this.postRepository.find({
      where: { writer: { id: userId } },
      relations: ['writer'],
      order: { createdAt: 'DESC', id: 'DESC' },
      skip: (page - 1) * limit,
      take: limit,
    });
    
    return result.map((post) => this.toPostResponseDto(post));
```
- skip ➡ offset 
- take ➡ limit 

#### 3.1.2.  QueryBuilder 버전

```js
    const result = await this.postRepository
      .createQueryBuilder('p')
      .where('p.writer_id = :userId', { userId })
      .orderBy('p.created_at', 'DESC')
      .addOrderBy('p.id', 'DESC')
      .skip((page - 1) * limit)
      .take(limit)
      .re
      .getManyAndCount();
      
    const [posts, total] = result;      
```

- Note : `QueryBuilder`에서는 `relations` 옵션이 안 먹는다
- 만약 가져오고 싶다면 `leftJoinAndSelect('p.writer', 'w')`로 가져와야 함 

```js
[
	// entitis 
  [
    Post {
      id: 'a42722c8-2390-4b20-87ec-6c2e19965aa5',
      createdAt: 2025-11-02T05:23:20.803Z,
      title: '테스트 제목21',
      content: '테스트 내용입니다',
      status: '모집중',
      startDate: '2015-01-01',
      endDate: '2016-01-01',
      location: '부산ㅇ',
      keywords: [Array],
      writer: undefined
    },
    Post {
      id: 'abbecd53-524e-4f3f-8027-854a38fd6ecc',
      createdAt: 2025-11-02T05:23:18.228Z,
      .....
      writer: undefined
    }
  ],
  17  // 이게 total 
]
```

`getManyAndCount` 구조 
```js
    getManyAndCount(): Promise<[Entity[], number]>;
    private lazyCount;
```


### 3.2.  offset 기반 Pagination

