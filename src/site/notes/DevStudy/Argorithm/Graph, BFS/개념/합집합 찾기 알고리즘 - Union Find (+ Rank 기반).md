---
{"dg-publish":true,"permalink":"/DevStudy/Argorithm/Graph, BFS/개념/합집합 찾기 알고리즘 - Union Find (+ Rank 기반)/","noteIcon":"","created":"2025-12-03T14:52:52.158+09:00","updated":"2025-12-13T18:25:27.364+09:00"}
---


**활용 트리 : [[DevStudy/Argorithm/Graph, BFS/개념/최소 신장 트리\|최소 신장 트리]]**

합집합 찾기 = 서로소 찾기 

> 여러 개의 노드가 존재할 때, 두 개의 노드를 선택해서 현재 이 노드가 **서로 같은 그래프에 속하는지 판별**하는 알고리즘 

#### 연결되어 있지 않을 때의 배열 
![Pasted image 20250725132318.png](/img/user/supporter/image/Pasted%20image%2020250725132318.png)
- 사진과 같이 노드가 자유롭게 존재하고 모두 연결되어 있지 않을 때 배열로 표현할 수 있다.
- **배열의 의미**
	- **첫 번째 행** : 노드 번호
	- **두 번째 행** : 부모 노드 번호 
- 각 노드는 자기 자신만 가리키므로 부모 노드가 자기 자신이다.

#### 연결 시 배열 

>[!QUESTION] 연결 시 어떻게 표현해야 할까❓
>- 가령, 1-2 가 연결되어 있다면 더 작은 값을 부모로 가지도록 하는 방법이 있다.
>- 여러 노드와 연결되어 부모가 여러개 일 수 있더라도 일반적으로 **더 작은 값 쪽으로 합친다(Union)**

![Pasted image 20250725134356.png](/img/user/supporter/image/Pasted%20image%2020250725134356.png)
- 근데 1-2-3 이런 구조로 연결되어 있더라도 1-3은 연결되어 있는 것이다. 
- 따라서 3의 부모는 1이 되는 것이 맞다. 이런 것을 표현하기 위해 프로그램에서는 "재귀 함수"를 활용한다
- 3의 부모를 찾는 재귀 함수 로직 예시 
	- 3이 가리키고 있는 2를 확인
	- 2의 부모 확인 ➡ 1 
	- 1의 부모 확인 ➡ 1  ⭐ 자기 자신을 가리키므로 얘가 부모다!

### 구현 

> 특정 값들 a, b 가 연결되어 있는지 확인하는 로직 


```python
# 부모 노드 찾기
def getParent(parents, x):
  if parents[x] != x:
		# 루트 노드를 찾기 위해 재귀적으로 호출
    parents[x] = findParent(parents, parents[x])
  return parents[x]

# a - b 노드를 연결하는 것
def unionParent(parent: list, a, b):
  a = getParent(parent, a)
  b = getParent(parent, b)
  if a = b: 
	  return False  # 합치기도 전에 부모가 같다는 것은 이미 연결되어있다는 뜻 
 
  if a < b: parent[b] = a
  else: parent[a] = b
  return True 
  
def findParent(parent, x):
  return parent[x]
  
if __name__ == '__main__':
  N = 8
  parent = [0] * N
  for i in range(1, N):
    parent[i] = i

  unionParent(parent, 1, 2)
  unionParent(parent, 2, 3)
  unionParent(parent, 4, 5)
  unionParent(parent, 5, 6)
  unionParent(parent, 6, 7)
  print(findParent(parent, 7))
```


>[!EXAMPLE] findParent()재귀 - 경로 압축 이해하기 
>- `1 ➡ 2 ➡ 3 ➡ 4`로 이루어진 트리가 있다고 가정하자
>	- 1의 부모는 2 
>	- 2의 부모는 3
>	- 3의 부모는 4
>- `findParent(parents, 1)` 을 호출하면 
>	- `parents[1] = findParents(parents, parents{1]) # (parents, 2)`
>	- `parents[2] = findParents(parents, parents{2]) # (parents, 3)`
>	- `parents[3] = findParenst(parents, parents[3]) # (parents, 4)`
>- 원래 parents[1], parents[2]는 2, 3 이였는데 findParents()를 호출하니 루트 노드인 4로 변하게 됨 
>- 이로 인해, **다음 호출 시**에는  `1 ➡ 2 ➡ 3 ➡ 4` 로 갈 필요 없이 `1 ➡ 4` 로 **바로 루트 노드를 찾을 수 있다.**
>- 이것이 바로 "**경로 압축**"


### Rank 기반 

> [!WARNING] 단순 재귀 호출 경로 압축의 단점 
> - 한 방향으로 연결되어 있을 경우 find(x)시 재귀 깊이가 길어질 수 있다.
> - 경로 압축 전에 이러한 문제 발생 

> 이에 대한 해결책이 "랭크 기반 Union이다"

#### 개념 
- **랭크(Rank) : 집합의 최대 트리 깊이 추정치** 
	- 박스 큰거 아래로 놓듯이 
	- 깊이가 높은(rank가 높은) 노드 밑에 랭크가 낮은 트리를 놓으면 불안정해짐 Cuz 트리 길이가 너무 늘어남  
- **병합 시 트리를 가능한 한 낮게 유지** ➡ 재귀 깊이를 줄이고 성능 향상 
	- 이전에 단순 재귀 호출 방식에서는 find 시 재귀 깊이를 줄이는 방식이였는데, 랭크 기반에서는 애초에 **병합 단계에서 재귀 깊이를 줄이는 작업**을 하는 것이다.,

#### 흐름 설명 
![Pasted image 20250725172357.png](/img/user/supporter/image/Pasted%20image%2020250725172357.png)
- Basic 코드에서는 노드의 숫자를 비교하면서 부모 노드를 만들었었다.
- 하지만 Rank기반에서는 **Rank기반으로 부모 노드를 결정**한다
- **Rank가 낮다면 높은 Rank의 root_node를 부모로 갖는다.**
- **❓Rank는 어떻게 하면 오르는가❓**
	- 동일 수준의 Ranker를 만나면 싸워서 누군가가 이겨야 되는데 이긴 애가 rank++


```python
parents = [1, 90, 91, 92, 93, 94, 95]
rank = [0, 0, 0, 0, 0, 0, 0]
```
- rank가 같을 경우에만 rank++
![Pasted image 20250725163046.png](/img/user/supporter/image/Pasted%20image%2020250725163046.png)


> Union(1, 95) 시 

![Pasted image 20250725163658.png](/img/user/supporter/image/Pasted%20image%2020250725163658.png)



### 추가 예시 (Basic vs Rank 기반)
> 사진과 같이 1-2번 노드가 연결되어 있고 8 ➡ 7 ➡ 4 ➡ 3 으로 연결되어 있는 상황에서 2번 노드와 8번 노드를 연결한다고 가정하자 <br>

![Pasted image 20250725170223.png](/img/user/supporter/image/Pasted%20image%2020250725170223.png)

#### Basic
> 단순히, 두 개의 트리 중 하나를 위에 다가 쌓는 방식 
> 자세한 설명은 사진 참고 
![Pasted image 20250725170430.png](/img/user/supporter/image/Pasted%20image%2020250725170430.png)

![Pasted image 20250725170517.png](/img/user/supporter/image/Pasted%20image%2020250725170517.png)

#### Rank 기반

![Pasted image 20250725172900.png](/img/user/supporter/image/Pasted%20image%2020250725172900.png)

![Pasted image 20250725172942.png](/img/user/supporter/image/Pasted%20image%2020250725172942.png)


 