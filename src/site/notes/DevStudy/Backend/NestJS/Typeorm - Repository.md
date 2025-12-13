---
{"dg-publish":true,"permalink":"/DevStudy/Backend/NestJS/Typeorm - Repository/","noteIcon":"","created":"2025-12-03T14:52:49.586+09:00","updated":"2025-12-13T10:30:55.877+09:00"}
---



NestJS에서는 TypeORM과 통할할 때 Repository를 생성하지 않아도 자동으로 주입받을 수 있다


**Repository 자동 생성** 
- 엔티티 User가 있으면 TypeORM은 내부적으로 `Repository<User>`를 자동 생성함 

## 1.  Repository 사용법

> 특정 엔티티를 TypeOrmModule에 등록해야 함 

### 1.1.  모듈에 등록 
```ts
@Module({
  imports: [TypeOrmModule.forFeature([User])], // User Repository 등록
  ...
})

export class UserModule {}
```


### 1.2.  Service에서 주입받기 

> `@InjectRepository(User)`, `Repository<User>` 사용 

```TS
@Injectable()
export class UserService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {}
```

