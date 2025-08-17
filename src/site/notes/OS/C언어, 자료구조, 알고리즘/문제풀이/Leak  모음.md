---
{"dg-publish":true,"permalink":"/OS/C언어, 자료구조, 알고리즘/문제풀이/Leak  모음/","noteIcon":"","created":"2025-08-18T00:58:47.005+09:00","updated":"2025-08-18T01:06:06.456+09:00"}
---



### Leak  모음 

1. **할당 후 null 체크를 안한 경우**
	```c
		❌부족한 예시 
		Queue *queue = (Queue *) malloc(sizeof(Queue));
	
		✅올바른 예시
	   Queue *queue = (Queue *) malloc(sizeof(Queue));
	   if (queue == NULL) return;
	```
	- 메모리가 제대로 할당이 됐는지 안됐는지 체크를 해야한다.

2. **메모리 해제를 안하는 습관**
	- 1번 문제를 푸는데 BFS를 구현하면서 Queue를 만들고 해제를 안했었따.
	```c
		 // ❌ 잘못된 경우 
		 Queue *queue = (Queue *) malloc(sizeof(Queue));
	   if (queue == NULL) return;
	
		 While ( ~~ ){
			 ~~~
		 }
	
		끝 

		✅올바른 경우 
		Queue *queue = (Queue *) malloc(sizeof(Queue));
		...
	
		While (!isEmpty(queue->head)){
		  BSTNode *node = dequeue(&queue->head, &queue->tail);
			....
      
      printf("%d", node->item);
      // 해제 1 
			free(node);

		// 해제 2 
		free(queue);
	```

3. **초기화 안 해주는 습관**
	- 구조체를 만들면 그 안의 속성들의 값이 반드시 Null이 아닐 수 있다.(완전 랜덤한 메모리 주소)
	- 따라서, 구조체를 만들고 속성 값들을 아래처럼 **초기화 시켜주는 습관이 중요**하다
		```c
		   Queue *queue = (Queue *) malloc(sizeof(Queue));
		   ...
		   queue->head = NULL;
		   queue->tail = NULL;
		```








