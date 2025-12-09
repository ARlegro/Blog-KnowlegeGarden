---
{"dg-publish":true,"permalink":"/Computer_Science/C언어, 자료구조, 알고리즘/문제풀이/LinkedList_solution/","noteIcon":"","created":"2025-12-03T14:52:45.748+09:00","updated":"2025-12-09T17:19:41.616+09:00"}
---



> 크래프톤 정글 Week 05 에서 있던 미션
> - C로 LinkedList 구현하기 


## Q3_A - 홀수를 뒤로 보내는 문제 
꼬리를 만들어간다? 라고만 생각하고 

미션은 주어진 연결리스트에서 홀수를 뒤로 보내는 미션이였다.
나는 처음에 돌면서 짝수를 앞으로 보내는 방식을 구현하려 했지만, 그 방식을 구현도중 팀원분과 다른 동료들이 꼬리를 만들었다는 것을 엿들어서 아? 내가 생각하는 방법이 아닌가?하는 마음에 꼬리를 먼저 찾고 다시 앞에서부터 순회하면서 홀수로 꼬리를 갱신하는 방식을 구현해봣는데 너무 어려웠다. (c자체도 처음이라 ㅠ)
처음에 내가 **의식의 흐름대로 짠 코드**는 아래와 같다. (💢 혐 주의)
```c 
void moveOddItemsToBack(LinkedList *ll)
{
  /* add your code here */
  ListNode *cur_node = ll->head;
  ListNode *tail_node = NULL;

  // 꼬리 찾기
  while (tail_node == NULL)
  {
    if (cur_node->next == NULL){
      tail_node = cur_node;
    } else{
      cur_node = cur_node->next;
    }
  }

  // 다시 처음부터
  cur_node = ll->head;
  ListNode *tail_p_node = tail_node;
  while (tail_p_node != NULL)
  {
    // 헤드 노드가 홀수인 경우
    if (ll->head == cur_node && cur_node->item & 1){
      ListNode *odd_node = cur_node;
      tail_node->next = odd_node;
      odd_node->next = NULL;
      ll->head = cur_node->next;
      cur_node = ll->head;
      continue;
    }

    // 현재 노드의 다음 노드가 홀수일 경우
    if (cur_node->item % 2 == 0 && cur_node->next->item & 1){
      ListNode *odd_node = cur_node->next;
      cur_node->next = odd_node->next; // 현재노드(짝수)의 다음 포인터를 홀수 노드의 다음 포인터로
			// 홀수 포인터 옮기기 💢
      tail_node->next = odd_node;
      odd_node->next = NULL;
    } else {
      // 끝까지 도달햇을 경우
      if (cur_node == tail_p_node) {
        tail_p_node = NULL;
        break;
      }
      cur_node = cur_node->next;  
    }
```
>가장 치명적인 문제는 `tail_node->next = odd_node;`이 부분에서 예상치 못한 순환이 발생한다는 것이다.
>- 저 코드를 짠 이유는 현재 노드는 짝수고 다음 노드가 홀수일 때 홀수는 뒤로 보내고 짝수 노드부터 다시 시작하려고 짰다.
>- 하지만 리스트가 만약 `[2,3]`일 때 아무리 3을 뒤로 보내도 계속 그대로이고 2도 그대로이기 때문에 무한루프가 발생하게 된다.


### GPT 답 

#### 버전 1 

```C
void moveOddItemsToBack(LinkedList *ll)
{
  /* add your code here */
  if (ll == NULL || ll->head == NULL || ll->size < 1){
    return;
  }

	// 초기 꼬리 찾기 
  ListNode *tail = ll->head;
  while (tail->next != NULL){
    tail = tail->next;
  }

  ListNode *original_tail_node = tail;
  ListNode *cur = ll->head;
  ListNode *prev = NULL;

  int isLopped = 0;
  while (!isLopped)
  {
    // 종료조건 = 한바퀴 다 돌았을 때,
    if (cur == original_tail_node){
      isLopped = 1;
    }

    // 현재 노드가 홀수인 경우
    if (cur->item & 1) {
      // 마지막일 경우 진행 x
      if (cur == tail) {
        break;
      }

      // 홀수를 담기
      ListNode *odd_node = cur;
      if (cur == ll->head) { // 헤드가 홀수인 경우
        ll->head = cur->next;
        cur = ll->head;
      } else { // 중간 홀수인 경우
        prev->next = cur->next;
        cur = prev->next;
      }

      tail->next = odd_node;
      tail = odd_node;
      odd_node->next = NULL;
    } else {
      prev = cur;
      cur = cur->next;
    }
  }
```
- 홀수 노드를 찾는데, prev 노드와 cur 노드를 저장해놓으면서 순회를 한다.
- 만약 cur노드가 홀수이면 prev노드의 next를 cur노드의 next로 연결시키고 cur노드는 tail로 가는 전체적 로직이다.
- 만약 cur노드가 짝수이면 prev = cur로 하고 cur을 다음 노드로 변경함으로써 진행
- **주의할 점** - `[2,3]` 예시 
	- `if (cur == tail)` 처리를 하지 않으면 `prev->next = cur->next;``cur = prev->next;` 부분에서 큰 문제가 발생한다.
	- 마지막 원소에서 저 로직을 발동시키면 `cur->next`는 `NULL`일 것이다. 근데 저 로직이 있으면 이전 원소(`2)`의 next가 `NULL`이 되어 오류가 발생하게 된다


#### V2 - 분할 후 병합 (나중에 분석)

```C
void moveOddItemsToBack(LinkedList *ll)
{
    // 리스트가 비어있거나 노드가 하나만 있으면 함수를 종료
    if (ll == NULL || ll->head == NULL || ll->size <= 1) {
        return;
    }

    // 짝수 노드와 홀수 노드를 관리할 두 개의 새로운 리스트 헤더
    ListNode *even_head = NULL;
    ListNode *even_tail = NULL;
    ListNode *odd_head = NULL;
    ListNode *odd_tail = NULL;

    ListNode *current = ll->head;
    ListNode *next_node;

    // 원래 리스트를 순회하며 짝수와 홀수 리스트로 분리
    while (current != NULL) {
        next_node = current->next; // 다음 노드를 미리 저장
        current->next = NULL;      // 현재 노드를 리스트에서 분리

        if (current->item % 2 == 0) { // 아이템이 짝수인 경우
            if (even_head == NULL) {
                even_head = current;
                even_tail = current;
            } else {
                even_tail->next = current;
                even_tail = current;
            }
        } else { // 아이템이 홀수인 경우
            if (odd_head == NULL) {
                odd_head = current;
                odd_tail = current;
            } else {
                odd_tail->next = current;
                odd_tail = current;
            }
        }
        current = next_node; // 저장해둔 다음 노드로 이동
    }

    // 분리된 두 리스트를 다시 합침
    if (even_head == NULL) {
        // 모든 노드가 홀수였을 경우
        ll->head = odd_head;
    } else {
        // 짝수 리스트의 끝에 홀수 리스트를 연결
        even_tail->next = odd_head;
        ll->head = even_head;
    }
}
```
분리시키는 법  `current->next = NULL; ` - 현재 노드를 리스트에서 분리


더 가독성 있다.



## Q7_A - 재귀호출을 사용해서 역순으로 만드는 문제 

### 문제 
- 연결리스트의 첫 원소의 주소를 `RecursiveReverse`라는 함수에 넘겨준다.
- 그 함수로 재귀를 구현해서 역순을 구현하게 만드는 문제

```c
void RecursiveReverse(ListNode **ptrHead){
	// add my code herer;
```

### 시도 

아이디어는 사진과 같았다. head부터 오른쪽으로 순회하면서 노드를 앞쪽으로 보내게 하는 로직이었다.
![Pasted image 20250809155914.png](/img/user/supporter/image/Pasted%20image%2020250809155914.png)

> 아래의 코드는 잘못된 코드이다.

```c
void RecursiveReverse(ListNode **ptrHead)
{
  /* add your code here */
  if (*ptrHead == NULL || (*ptrHead)->next == NULL) {
    return;
  }
  ListNode *nextNode = (*ptrHead)->next;
  ListNode *curNode = (*ptrHead);

  // 다음 노드 가리키도록
  curNode->next = nextNode->next;

  // 💢문제 발생 : 원래 다음 노드였던 노드는 curNode를 가리지도록
  nextNode->next = curNode;
  RecursiveReverse(*ptrHead);
}
```
- **문제 발생 코드 - `nextNode->next = curNode`**
	- 저렇게 하면 앞으로 가는 모든 노드들이 사진기준 1을 바라보게 된다.
![Pasted image 20250809161756.png](/img/user/supporter/image/Pasted%20image%2020250809161756.png)

>[!QUESTION]  고민 - 전역변수를 둬야 하나❓ 파라미터를 늘리면 안되나❓
>- 재귀함수를 만들 때 그냥 필요하면 파라미터를 늘렸어서 고민을 좀 했었다.
>- 근데 문제의 의도는 정해진 파라미터로만(전역변수도 없이) 풀라는 의도 같았다.
>- 도대체 어떻게 풀어야할까? 라고 고민을 하다가 답을 보게 되었다.


### 답 
```c
void RecursiveReverse(ListNode **ptrHead)
{
  /* add your code here */
  if (*ptrHead == NULL || (*ptrHead)->next == NULL) {
    return;
  }
  ListNode *first = (*ptrHead);
  ListNode *rest =  (*ptrHead)->next;
  
  RecursiveReverse(&rest);
  // 뒤집힌 꼬리의 끝을 parentNode로 연결
  (*ptrHead)->next->next = first;
  
  // parentNode는 이제 꼬리이므로 next = null
  (*ptrHead)->next = NULL;
  
  // 함수로 전달받은 head 교체  
  // 이유 : 이거를 안하면 반환 후 이전 호출의 (*ptrHead)->next 에서 null이 나옴
  *ptrHead = rest;
}
```
**원리 - 분할 정복 원리** 
- 분할 : 현재 헤드와 나머지부분(rest)로 분리한다.
- 정복 : 재귀를 통해 rest를 끝까지 뒤집으면 rest는 뒤집힌 부분리스트의 꼬리가 된다.
- 이 꼬리의 next를 현재 헤드로 교체한다.


**풀어본 썰** 
> 헤드 포인터 변경 전파에 실패한 썰 
- 처음에 답을 보고 30분 뒤에 다시 풀어봤는데 계쏙 오류가 났었다.
- 원인은 `*ptrHead = childNode;`이 부분을 놓치고 있었기 때문이다.
- `*ptrHead = childNode;`를 안한다면 `ptrHead`를 호출한 콜러의 다음 로직인 `(*ptrHead)->next`에서 null이 나타나서 버그가 발생했던 것이다.
- `*ptrHead = childNode;`를 한다면  `(*ptrHead)->next`가 null이 아니게 되어서 예상되로 역순 로직이 만들어지게 된다.

