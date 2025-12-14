---
{"dg-publish":true,"permalink":"/DevStudy/Argorithm/Graph, BFS/문제풀이/백준 1260 - DFS and BFS/","noteIcon":"","created":"2025-12-03T14:52:52.073+09:00","updated":"2025-12-13T18:25:27.390+09:00"}
---



>**맞췄는가?  Yes**

### 리뷰 이유 
- 내 코드의 성능보다 8배 빠른(420ms ➡ 60ms) 사람이 있길래 그거 습득하려고 리뷰 


### 문제 링크 
https://www.acmicpc.net/problem/1260


### 나의 답 ✅

- **인접 행렬 그래프**를 활용해서 연결 여부를 확인했다. (참고 링크 : [[DevStudy/Argorithm/Graph, BFS/개념/Graph\|Graph]]) 
- **한번 들린 곳의 좌표는 0으로 처리**해서 visited 효과
	- 무방향 그래프이기 때문에 대칭으로 0처리해야 한다. 
- bfs, dfs는 사실 week 01 에서 다뤘던 주제라 크게 설명할 것이 없다.

```python
import sys
from collections import deque
input = sys.stdin.readline

# DFS 
def dfs(graph, root: int, result: list):

	N = len(graph)
  for i in range(1,N):
    var = graph[root][i]
    if not var or i in result:
      continue

    result.append(i)
    # 방문한 곳은 0 처리 
    graph[root][i] = 0
    graph[i][root] = 0
    # 재귀 호출 
    dfs(graph, i, result)

def bfs(graph, root, result: list):
  q = deque()
  q.append(root)

  while q:
    popped = q.popleft()
    for i in range(1, len(graph)):
      var = graph[popped][i]
      if not var or i in result:
        continue

			# 방문한 곳은 0 처리 (대칭)
      graph[popped][i] = 0
      graph[i][popped] = 0
      result.append(i)
     
	    # FIFO 큐에 담기 Cuz BFS 
      q.append(i)

if __name__ == '__main__':
  N, M, V = map(int, input().strip().split())
  dfs_graph = [[0]*(N+1) for _ in range(N + 1)]
  bfs_graph = [[0]*(N+1) for _ in range(N + 1)]
  for _ in range(M):
    u, v = map(int, input().split())
    # 연결된 부분 1 처리 
    dfs_graph[u][v] = 1
    dfs_graph[v][u] = 1
    bfs_graph[u][v] = 1
    bfs_graph[v][u] = 1

  dfs_result = [V]
  bfs_result = [V]
  dfs(dfs_graph, V, dfs_result)
  bfs(bfs_graph, V, bfs_result)

  print(*dfs_result)
  print(*bfs_result)
```

일단 내 코드를 리뷰하기 전에 다른 사람의 코드를 보고 내 코드가 얼마나 비효율적인지 파악해보자 


### 다른 사람 코드 

- **인접 리스트**를 사용했다.
	- 아래와 같은 장점이 있다.
	- DFS, BFS 에서는 **특정 노드에 연결된 노드를 탐색하는 것이 중요하므로 인접 리스트가 좋은** 것 같다 
>[!tip] 인접 리스트의 장단점 in [[DevStudy/Argorithm/Graph, BFS/개념/Graph\|Graph]]
>1. 장점 ✅
>	- **메모리를 덜 씀** : 실제로 연결된 노드만 저장함. 따라서, 간선이 적을 때 효율적 .
>	- **간선 순회가 빠르다** : 특정 노드에 **연결된 노드만 바로 탐색**하므로 **DFS, BFS 등 그래프 순회 시 효율적** 
>	- **동적 그래프에 유리** : 간선 추가/삭제 시 append, remove만 하면 됨 
>	  
>2. 단점 💢
>	- **간선이 많은 그래프에서는 불리**
>		- 간선이 많은 경우 리스트 순회 비용이 커져서 조회/수정 부담 
>		- 오히려 이 경우는 인접 행렬이 더 나을 수 있음 
>	- **연결 여부 확인 시 느리다**
>		- 인접 행렬 : $O(1)$
>		- 인접 리스트 : $O(연결된 노드 수)$

- **visited 배열을 따로 만들었다.**

```PYTHON
import sys
from collections import deque
input = sys.stdin.readline
n, m, v = map(int, input().split())

graph = [[] for _ in range(n+1)]
visited = [False]*(n+1)

def dfs(x):
    visited[x] = True
    print(f"{x}", end=' ')
    for y in graph[x]:
        if not visited[y]:
            dfs(y)

def bfs(start):
    q = deque()
    q.append(start)
    visited[start] = True
    while(q):
        x = q.popleft()
        print(f"{x}", end=' ')
        for y in graph[x]:
            if not visited[y]:
                q.append(y)
                visited[y] = True

for _ in range(m):
    x, y = map(int, input().split())
    graph[x].append(y)
    graph[y].append(x)

for abj in graph:
    abj.sort()

dfs(v)
visited = [False]*(n+1)
print()
bfs(v)
```

#### DFS, BFS 부분 차이 (나 vs 상대)
![Pasted image 20250725232512.png](/img/user/supporter/image/Pasted%20image%2020250725232512.png)
![Pasted image 20250725233009.png](/img/user/supporter/image/Pasted%20image%2020250725233009.png)
- **순회 부분 차이** 
	- **나** : 해당 행의 그래프르 전부 순회해야 함 
	- **다른** : 해당 값이 가지는 리스트 값만 조회하면 됨
- **방문 체크 차이** 
	- **나** : graph에 양방향 표시 
	- **다른** : visited에 표시. 애초에 **방문이 되어 있으면 재귀 호출을 부르지 않아서 효율적**  ⭐⭐

#### SORT 부분 디테일 

```PYTHON
for abj in graph:
    abj.sort()
```
- **문제 요구사항** : 연결된 간선이 여러 개일 경우 작은 순서대로 방문 


### 내 식대로 카피 
![Pasted image 20250725235449.png](/img/user/supporter/image/Pasted%20image%2020250725235449.png)
```PYTHON
import sys
from collections import deque
input = sys.stdin.readline

graph = []
visited = []

def dfs(x: int, result: list):
  visited[x] = True
  result.append(x)
  # graph[x]를 탐색한다.
  for var in graph[x]:
    if not visited[var]:
      dfs(var, result)      

def bfs(V: int, result: list):
  q = deque()
  q.append(V)
  visited[V] = True
  result.append(V)

  while(q):
    popped = q.popleft()
    for var in graph[popped]:
      if not visited[var]:
        q.append(var)
        result.append(var)
        visited[var] = True

if __name__ == '__main__':
  N, M, V = map(int, input().strip().split())
  graph = [[] for _ in range(N + 1)]
  visited = [False] * (N+1)
  for _ in range(M):
    u, W = map(int, input().split())
    graph[u].append(W)
    graph[W].append(u)

  for g in graph:
    g.sort()

  dfs_result = []
  dfs(V, dfs_result)
  visited = [False] * (N+1)
  bfs_result = []
  bfs(V, bfs_result)
  sys.stdout.write(" ".join(map(str, dfs_result)) + "\n")
  sys.stdout.write(" ".join(map(str, bfs_result)) + "\n")
```

