---
{"dg-publish":true,"permalink":"/DevStudy/DB/Fundamentals of DB engineering/Concurrency Control/Deep dive Postgre's Isolatition Behavior/","noteIcon":"","created":"2025-12-03T14:52:51.158+09:00","updated":"2025-12-13T10:56:55.700+09:00"}
---



>[!tip] 이걸 이해해야, 왜 READ COMMITED에서 충돌이 잘 안 나는지를 이해할 수 있는 것 같다.
>- 아래는 PostgreSQL공식 문서에 써져있는 것을 해석한 것. 
>- https://www.postgresql.org/docs/current/transaction-iso.html#XACT-READ-COMMITTED


## 1.  13.2.1. Read Committed Isolation Level


_Read Committed_ is the default isolation level in PostgreSQL. 

When a transaction uses this isolation level, a `SELECT` query (without a `FOR UPDATE/SHARE` clause) sees only data committed before the query began; **it never sees either uncommitted data or changes committed by concurrent transactions during the query's execution.** In effect, **a `SELECT` query sees a snapshot** of the database as of the instant the query begins to run. 

*자신의 트랜잭션 내 UnCommited 수정 내용은 볼 수 있음*
However, `SELECT` does see the effects of previous updates executed within its own transaction, even though they are not yet committed. 

*SnapShot을 사용하지만 같은 트랜잭션 내 2개의 SELECT문이 다른 데이터를 볼 수 있다 When 중간에 다른 트랜잭션이 변화를 주고 commit했을 때(여전히  UnReapeatable Read 문제 💢)* 
Also note that two successive `SELECT` commands can see different data, even though they are within a single transaction, **if other transactions commit changes** after the first `SELECT` starts and before the second `SELECT` starts.

### 1.1.  Refresh 원리 (재평가 원리) 
*아래의 명령들은 이전 문장에서 설명한 SELECT 처럼 해동한다*
*무슨 말이야? 명령 시작시점에 커밋된 대상 행만 찾는다는 것* ⭐⭐⭐ <<< 이게 중요 개념 
`UPDATE`, `DELETE`, `SELECT FOR UPDATE`, and `SELECT FOR SHARE` commands **behave the same as `SELECT`** in terms of searching for target rows: they will only find target rows that were committed **as of the command start time.** 

*위의 명령들로 target Row를 찾았을 때 락걸려있으면 대기에 빠짐*
However, such a target row might have already been updated (or deleted or locked) by another concurrent transaction by the time it is found. In this case, the would-be updater **will wait** for the first updating transaction to commit or roll back (if it is still in progress). 

*락걸은 트랜잭션이 롤백되면 ➡ 대기중인 트랜잭션은 정상 진행*
If the first updater rolls back, then its effects are negated and the second updater can proceed with updating the originally found row.

*락걸은 트랜잭션이 해당 로우를 삭제 후 커밋하면 ➡ 두번째 트랜잭션은 해당행을 무시(UPDATE 0 표시)*
If the first updater commits, the second updater will ignore the row if the first updater deleted it, 

*락걸은 트랜잭션이 해당 로우를 단순 수정 후 커밋하면 ➡ 두번째 트랜잭션은 **업데이트 된 row에 자신의 작업을 적용**하려고 시도할 것
이 때, 조건(WHERE 절)은 재평가(Re-Evaluated)되어 업데이트된 버전의 행이 여전히 검색 조건과 일치하는지 확인*⭐⭐⭐
otherwise it will attempt to **apply its operation to the updated version of the row.** The search condition of the command (the `WHERE` clause) is re-evaluated to see if the updated version of the row still matches the search condition. If so, the second updater proceeds with its operation using the updated version of the row. In the case of `SELECT FOR UPDATE` and `SELECT FOR SHARE`, this means it is the updated version of the row that is locked and returned to the client.


## 2.  "일관되지 않은 스냅샷" ⭐

### 2.1.  개념 
`UPDATE`나 `DELETE` 명령은 같은 쓰기 명령이 **자신이 업데이트하거나 삭제하려는 동일한 특정 행에 대해서는 다른 동시 업데이트 명령의 영향을 볼 수 있지만, 데이터베이스 내의 다른 행에 대한 해당 명령들의 영향은 보지 못한다** ⭐
Because of the above rules, it is possible for an updating command to see an inconsistent snapshot: it can see the effects of concurrent updating commands on the same rows it is trying to update, but it does not see effects of those commands on other rows in the database. 

> 이는 하나의 명령 안에서 **부분적인 데이터는 최신 커밋된 버전을 따르지만, 전체 데이터베이스 뷰는 명령 시작 시점의 스냅샷에 묶여 불일치가 발생**할 수 있다는 의미
> - 이러한 동작은 `READ COMMITTED` 모드가 복잡한 검색 조건을 포함하는 명령에는 적합하지 않을 수 있음을 의미
> - 그러나 단순한 경우라면 문제없다.

### 2.2.  복잡한 경우 : DELETE 명령의 예상치 못한 결과
복잡한 시나리오에서는 `READ COMMITTED` 모드가 예상치 못한 결과를 초래할 수 있다.
More complex usage can produce undesirable results in Read Committed mode. For example, consider a `DELETE` command operating on data that is being both added and removed from its restriction criteria by another command, e.g., 
```sql
(초기 값)
select * from website;
 id | hits 
----+------
  1 |    9
  2 |   10
(2 rows)

----
-- T1 
BEGIN;

-- T2;
BEGIN;

-- T1 UPDATE 
UPDATE website SET hits = hits + 1;

-- T2 DELETE
DELETE FROM website WHERE hits = 10; -- 락걸린 상태

-- T1 COMMIT;
COMMIT;

-- T2 락 풀림. 과연 제거가 됐을까❓
-- 처음 hits가 10인 rows는 id=-2인 row이다.
-- 락대기가 풀리고 업데이트 된 버전으로 봐도 id=1인 row가 hits=10이니 삭제가 되야하지 않을까???

-- T2 DELETE 결과 : 아무것도 삭제되지 않았다. ⭐⭐⭐
DELETE FROM website WHERE hits = 10;
DELETE 0

SELECT * FROM website;
 id | hits 
----+------
  1 |   10
  2 |   11
(2 rows)

```
- `DELETE` 문은 내부적으로 **후보가 되는 행을 스캔**하여 조건에 부합하는 행에 락을 걸고 삭제 처리함
- `DELETE` 명령은 이미 후보를 결정했기 때문에 **새롭게 `hits=10`이 된 `id=1`을 다시 스캔하여 삭제하려하지 않는다**

### 2.3.  발생 원인 
>[!QUESTION] 왜 이런 "일관되지 않은 스냅샷 현상"이 발생하는가❓ ⭐⭐
> 1. 초기 후보 선택은 명령 스냅샷 기반
> 2. 락 해제 후 **재평가는 기존 후보 행에만 적용**
> 3. 새롭게 조건에 부합하게 된 행은 고려되지 않음

Because Read Committed mode starts each command with a new snapshot that includes all transactions committed up to that instant, subsequent commands in the same transaction will see the effects of the committed concurrent transaction in any case. The point at issue above is whether or not a _single_ command sees an absolutely consistent view of the database.

The partial transaction isolation provided by Read Committed mode is adequate for many applications, and this mode is fast and simple to use; however, it is not sufficient for all cases. Applications that do complex queries and updates might require a more rigorously consistent view of the database than Read Committed mode provides.

### 2.4.  해결 시도 

#### 2.4.1.  비관적 락 적용
![Pasted image 20250607141713.png](/img/user/supporter/image/Pasted%20image%2020250607141713.png)
- 어차피 READ COMMITED 수준에서 PostgreSQL의 동작은 비관적 락이랑 전혀 상관없다.
- 당연히 똑같은 오류 발생 

#### 2.4.2.  격리수준 격상 to REPEATABLE READ
![Pasted image 20250607142855.png](/img/user/supporter/image/Pasted%20image%2020250607142855.png)
- 대기중이던 DELETE가 ROLLBACK됐다.
- 이유 : PostgreSQL은 `REPETABLE READ`에서는 락 해제 후 target row가 다른 트랜잭션에 의해 update됐다면 해제된 트랜잭션은 ROLLBACK된다.  

>[!tip] 이 예시에서 UPDATE는 격리 수준 그냥 놔둬도 된다

참고 : [[DevStudy/DB/Fundamentals of DB engineering/Concurrency Control/MVCC\|MVCC]] -> 격리수준 격상으로 Lost Update 막기 in PostgreSQL




근데 이러면 DELETE를 계속 재시도 노려야하나❓
격리수준 격상 + 재시도 로직 ❓❓❓


## 3.  그렇다면 READ COMMITED 적합한가❓
✅
단순하고 예측 가능한 행에 대해서는 효율적이고 안전 
Cuz 락 해제 후 행의 최신 커밋 상태를 반영하여 `LOST UPDATE` 막아줌 

💢
그러나 앞서 언급한 "일관되지 않은 스냅샷" 현상은 복잡한 검색 조건, 동적 변경으로 인한 기준 부합 의 경우에는 **예측 불가능한 결과를 초래**할 수 있다.

ex. 10살의 데이터는 전부 DELETE 명령 ➡ 한 row가 10살로 UPDATE되는거를 막지 못했을 때, 데이터에는 10살의 데이터가 존재하는 문제 ❗
> 이럴 때는, 애플리케이션에서 더 엄격한 격리수준 고려 하거나 동시성을 더 세밀하게 제어 必
