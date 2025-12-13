---
{"dg-publish":true,"permalink":"/DevStudy/Backend/Redis/Add @CacheEvict + @CachePut/","noteIcon":"","created":"2025-12-03T14:52:50.186+09:00","updated":"2025-12-13T10:23:07.679+09:00"}
---



일반적으로 DB 트랜잭션을 배울 때, ACID 속성을 배웠을 것이다.
그 중 C(Consistency)는 분산 시스템 or 캐싱 아키텍쳐에서 문제가 드러난다.

자세한 설명은 ACID공부하면 알 것이니 각설하고 
Redis같은 캐싱기능을 사용하면 캐시 데이터와 DB데이터의 불일치가 발생할 수 있다.
왜냐하면, 수정작업을 캐싱이 바로 반영하지 못하기 때문이다.

이러한 문제를 해결해주는 기능을 Spring은 제공한다.
참고 : [[DevStudy/Backend/Spring/Cache/Basic Concept + Annotation (many use)\|Basic Concept + Annotation (many use)]]
1. @CacheEvict : 이 애노테이션이 붙은 메서드를 실행 후, 특정 캐시를 제거
2. @CachePut : 이 애노테이션이 붙은 메서드를 실행 후, 특정 캐시를 갱신 

> 이 애노테이션들의 자세한 내용은 참고 링크에 포함 
> 지금 이 페이지는 이것들과 Redis를 이용했을 때 실제로 차이가 나는지 확인용 


테스트용 메서드는 다음과 같다
```java
@Cacheable(cacheNames = "boards", key = "'board:page:'+ #page + ':size:' + #size", cacheManager = "boardCacheManager")  
public List<Board> getBoards(int page, int size) {  
    Pageable pageRequest = PageRequest.of(page - 1, size);  
    Page<Board> pageOfBoard = boardRepository.findAllByOrderByCreatedAtDesc(pageRequest);  
    return pageOfBoard.getContent();  
}  
  
@CacheEvict(cacheNames = "boards", key = "'board:page:1:size:10'", cacheManager = "boardCacheManager")  
public void update(Long id) {  
    Board board = boardRepository.findById(id)  
        .orElseThrow(() -> new RuntimeException("게시글이 존재하지 않습니다."));  
    board.setContent("수정된 내용");  
}
```
- getBoards : 캐싱기능이 있는 메서드 
- update : 수정 후 캐싱을 지우는 메서드 

> 참고 : @CacheEvict애노테이션 쓰지 않고 Redis 만 썼을 시 부하테스트 결과 ➡[[DevStudy/Backend/Redis/Load Testing\|Load Testing]]


Postman으로 getBoards메서드를 실행시켰을 때, 캐싱 기준 평균 6ms의 시간이 소요된다.
이러한 효과는 TTL까지 지속된다.
![Pasted image 20250605113333.png](/img/user/supporter/image/Pasted%20image%2020250605113333.png)
하지만 TTL이 지나지 않았음에도 @CacheEvict가 붙은 메서드를 실행시키면 어떻게 될까?
![Pasted image 20250605113452.png](/img/user/supporter/image/Pasted%20image%2020250605113452.png)
274ms 로 늘었났다.
TTL이 지나지 않았음에도 캐싱이 끝난 이유는 @CacheEvict로 인해 캐싱된 데이터가 지워졌기 때문이다.
따라서 GET요청 시 캐싱 저장소가 아닌 실제 Disk에 접근하기에 속도가 느려진 것 

캐싱을 공부하면, 데이터의 일관성과 속도 사이 TradeOff에 마주친다.
사실 @CacheEvict가 일관성을 지켜준다해도 만약, 수정이 많은 서비스라면❓캐시의 효과가 정말 미미해진다. 
결국 수정은 적고 조회가 많은 서비스에서 빛나는 것이 캐시인 것 















