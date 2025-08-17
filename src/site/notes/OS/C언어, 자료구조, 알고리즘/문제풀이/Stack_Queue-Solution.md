---
{"dg-publish":true,"permalink":"/OS/C언어, 자료구조, 알고리즘/문제풀이/Stack_Queue-Solution/","noteIcon":"","created":"2025-08-10T14:14:06.582+09:00","updated":"2025-08-18T01:05:54.012+09:00"}
---



**크래프톤 정글 Week5 C언어 구현 커리큘럼 중 스택 큐 관련 문제들을 C언어로 구현하는 문제였다.**
이전 LinkedList([[OS/C언어, 자료구조, 알고리즘/문제풀이/LinkedList_solution\|LinkedList_solution]]) 보다는 전체적으로 쉬워서 금방금방 끝났는데 몇 가지 아쉬웠던 점 때문에 리뷰를 하게 되었다.

## 리뷰1. Q5_C_SQ


### 문제 
> **큐의 원소들을 재귀 호출로 뒤집는** 문제였다.

(recursiveReverseQueue)
- Write a recursive C function recursiveReverseQueue() that reverses the order of items stored in a queue of integers. int temp;
- The function prototype is given as follows: `void recursiveReverseQueue(Queue *q);`
- For example, if the queue is (1, 2, 3, 4, 5), then the resulting queue will be (5, 4, 3, 2, 1).


### 고민 및 정답

- 정글 커리큘럼의 Week1 ~ Week4까지 알고리즘만 풀면서 구현능력을 키웠다보니 재귀 호출 구현은 어렵지 않았었다. 
- 근데 처음에 헷갈렸던게 기존 q가 아닌 새로운 곳에 넣는 방식을 구현해야 한다고 생각하다보니 고정된 인자 하나로 그게 구현이 되나? 하면서 고민에 빠졌다.
- 근데 생각해보면 굳이 그렇게 새로운 곳(새로운 Q)를 만들지 않아도  아래처럼 아주 간단히 구현이 가능했다.
- 재귀 호출을 C언어에서 보게되니 반가웠고, 이렇게 간단한거라도 나를 고민하게 만들었다는게 재밌던 경험이였다.
```c
void recursiveReverse(Queue *q)
{
	/* add your code here */
  if (q->ll.head == NULL) return;
  int temp = dequeue(q);
  recursiveReverse(q);
  enqueue(q, temp);
}
```
- `deque()`를 하고 다음 호출들이 로직들을 다 처리한 다음에 `enqueue()`

