---
{"dg-publish":true,"permalink":"/Computer_Science/C언어, 자료구조, 알고리즘/문제풀이/Binary_Tree_Search-Solution (4번 나중에 정리)/","noteIcon":"","created":"2025-12-03T14:52:45.758+09:00","updated":"2025-12-09T17:19:41.590+09:00"}
---


### 2번 문제 - 중위 순회 출력
![Pasted image 20250812200356.png](/img/user/supporter/image/Pasted%20image%2020250812200356.png)
> **반드시 스택 자료구조를 사용**해서 중위 순회하는 문제이다.(재귀 ❌)

주어진 구조체는 아래와 같다
```c
typedef struct _bstnode{
  int item;
  struct _bstnode *left;
  struct _bstnode *right;
} BSTNode;   // You should not change the definition of BSTNode


typedef struct _stackNode{
  BSTNode *data;
  struct _stackNode *next;
}StackNode; // You should not change the definition of StackNode

typedef struct _stack
{
  StackNode *top;
}Stack; // You should not change the definition of Stack
```

### 문제 발생 
이런 문제는 재귀에 익숙해서 스택으로 푸는 것이 감이 잘 안왔다,
한가지 확실한 건 좌측으로 쭉~ While문을 돌며 push한 다음에 천천히 pop하면서 올라오는 것이다.
근데 pop해서 올라올 때 왼쪽 부분 null체크하는 로직땜에 무한 루프가 돌게 되었다.
```c
// 문제의 일부 로직
   while (s->top != NULL || isEmpty(s))
   {
	   // ❌ 여기서 계속 걸림 
      while (cur->left != NULL)
      {
        push(s, cur);
        cur = cur->left;
      }

      BSTNode *popped = pop(s);
      printf("%d ", popped->item);
      if (popped->right != NULL){
        push(s, popped->right);
        continue;
      }
```
![Pasted image 20250812201004.png](/img/user/supporter/image/Pasted%20image%2020250812201004.png)


### GPT 정답 
> 1시간 정도 고민하다가 답을 보게되었다. 
> 4주 동안 알고리즘 공부를 하면서 구현력이 많이 늘었다고 생각했는데 아직 갈 길이 먼 것 같다.


핵심 = while문이 다시 시작하기 전에 우측으로 옮기는 로직이다.
```c
void inOrderTraversal(BSTNode *root)
{
  Stack s = {NULL};
  BSTNode *cur = root;
  while (cur != NULL || !isEmpty(&s))
  {
    // 왼쪽 쭉 탐색
    while (cur != NULL)
    {
      push(&s, cur);
      cur = cur->left;
    }
    cur = pop(&s);
    printf("%d ", cur->item);
    // 이게 핵심
    cur = cur->right;
 }
```


>[!EXAMPLE] 번외 : {NULL} 의미찾기 - 부분 초기화
>코드를 보면 `Stack s = {NULL};` 이렇게 자동할당하는 부분이 있다. 아직 C언어의 문법에 익숙하지 않기에 이 문법을 잠시 공부하기로 했다.
>- `{NULL}` 의미는 구조체의 첫 번째 멤버를 NULL로 초기화하겠다는 뜻이다.
>- 만약 구조체의 멤버 변수가 여러개여도 이렇게 한 변수를 선언시점에 초기화한다면 **나머지 값들은 자동으로 0비트 (0 or Null)로 초기화** 된다 ➡ 이것이 "**부분 초기화**"
>- *C언어는 자동 변수의 초기화 값은 선언 시점에 지정 가능*


하... 사실 진짜 한 끝 차이긴 했는데 왜 못 떠올렸을까...
오른쪽 노드들이 있으면 push해야 한다는 것도 알고 왼쪽으로 쭉 가야된다는 것도 아는데 아직 구현력이 부족한 것 같다. 마침 2번 말고도 3~5번까지 전부 BTS문제이던데 이 기회에 이런 패턴에 익숙해져보자 


### 문제 4 - 스택하나로 후위 순회 

> 단순 Stack이 아니라 LastVisited 포인터를 사용하여 부모 노드 방문 타이밍을 정확하게 판단해야 해서 까다로웠다.

마지막으로 방문한 노드를 확인해야 한다.



#### 핵심 : 배열 없이 lastVisited 활용
알고리즘에서 방문 여부를 확인할 때는 visited 배열을 만들어서 확인했었다.
하지만 이 문제는 뭐랄까... 배열 크기도 무한대로 정하기 좀 그래서 visited 배열을 사용할 수 없었다.





