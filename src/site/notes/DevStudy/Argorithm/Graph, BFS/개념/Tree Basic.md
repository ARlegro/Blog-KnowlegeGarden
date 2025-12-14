---
{"dg-publish":true,"permalink":"/DevStudy/Argorithm/Graph, BFS/개념/Tree Basic/","noteIcon":"","created":"2025-12-03T14:52:52.171+09:00","updated":"2025-12-13T18:25:27.319+09:00"}
---



#알고리즘 

> 트리 : 계층적인 구조를 표현할 때 사용할 수 있는 자료구조 

#그래프의한형태 

그래프 > 트리
### 관련 용어 
![Pasted image 20250724140714.png](/img/user/supporter/image/Pasted%20image%2020250724140714.png)
- **루트 노드** : 부모가 없는 최상위 노드 
- **리프(단말 or 외부) 노드** : 자식이 없는 노드 (보통 가장 아래쪽)
- **크기** : 모든 노드의 개수 
- **깊이** : 루트 노드로부터의 거리 
- **차수(degree)** : 각 노드의 **(자식 방향) 간선 개수** 

>[!tip] 트리가 N개 일 때, 전체 간선의 개수  = **N -1** 


### 이진 탐색 트리 

#### 개념 및 특징 
- **이진 탐색이 동작할 수 있도록** 고안된 자료구조 
- **특징** : '왼쪽 자식 노드' < '부모 노드' < '오른쪽 자식 노드'

#### 조회 방법 
> 루트 노드부터 방문하여 탐색 시작 (나머지는 이진 탐색 원리)
![Pasted image 20250724135856.png](/img/user/supporter/image/Pasted%20image%2020250724135856.png)

> [!WARNING] 일반 트리에서는 불가능하다 


### 트리의 순회 
> 노드를 특정한 방법으로 한번씩 방문하는 방법 

종류는 아래와 같이 3가지가 있다.
1. **전위 순회** : **루트** ➡ 왼쪽 ➡ 오른쪽  (bfs)
2. **중위 순회** :  왼쪽 ➡ **루트**➡ 오른쪽 (dfs)
3. **후위 순회**  :  왼쪽 ➡ 오른쪽 ➡ **루트**  (하위 트리부터 방문하게 됨)

### 이진 트리 구현하기 

> 아이디어 
> - 계속 인덱스 범위가 좁아지더라도 이진 탐색 시 로직은 똑같으니 **재귀함수 활용** 
> - 함수에 필요한 정보
> 	- 트리를 담을 배열 
> 	- 재귀로 처리할 데이터 그룹들 범위
> - 

1. 노드 클래스 만들기
2. 트리 클래스 만들기
3. 노드 삽입 
4. 순회?

```java
public class BinaryTree {  
    // 트리 만들기  
    static class Tree {  
        Node root;  
  
        // 배열 정보를 받아서 트리를 만드는 정보  
        public void makeTree(int[] a){  
            // 재귀로 Tree를 만듬  
            // 재귀가 끝나면 root Node를 받아서 멤버변수에 저장  
            root = makeTreeR(a, 0, a.length - 1);  
        }  
  
        public Node makeTreeR(int[] a, int start, int end) {  
            if (start > end) return null;  
            // 중간 값을 기준으로 새로운 노드를 생성  
            int mid = (start + end) / 2;  
            Node node = new Node(a[mid]);  
            node.left = makeTreeR(a, start, mid - 1);  
            node.right = makeTreeR(a, mid + 1, end);  
            return node;  
        }  
  
        //트리가 잘 만들어졌는지 확인하는 메서드  
        public void searchBTree(Node n, int find){  
            if (n.data < find){  
                System.out.println("Data is smaller than " + n.data);  
                searchBTree(n.right, find);  
            } else if (n.data > find) {  
                System.out.println("Data is bigger than " + n.data);  
                searchBTree(n.left, find);  
            } else {  
                System.out.println("Data founed");  
            }  
        }  
    }  
    // Node 클래스 만들기  
    static class Node {  
        int data; // 노드의 값  
        Node left;  
        Node right;  
        // 생성자에서 값을 받아서 data 저장  
        public Node(int data) {  
            this.data = data;  
        }  
    }  
  
    public static void main(String[] args) {  
        int[] a = new int[10];  
        for (int i = 0; i < a.length; i++) {  
            a[i] = i;  
        }  
        Tree tree = new Tree();  
        tree.makeTree(a);  
        tree.searchBTree(tree.root, 2);  
    }  
}
```


![Pasted image 20250724155554.png](/img/user/supporter/image/Pasted%20image%2020250724155554.png)
배열이 있을 때 트리 
- 가운데 인덱스의 값이 루트노드 (작은 쪽은 왼쪽, 큰쪽은 오른쪽)
- 작은 쪽의 중간 값을 루트 노드의 왼쪽 자식 값으로 
- 큰 쪽의 중간 값을 루트 노드의 오른쪽 자식 값으로 



### 순회 방법별 이진트리 구현 

#### 0. 공통 로직 
![Pasted image 20250724170841.png](/img/user/supporter/image/Pasted%20image%2020250724170841.png)
> Insert() 기반 트리라서 삽입 순서가 구조를 결정하는게 아쉽
> - 나중에 균형 이진트리 방버중간값 재귀 방법이 있다니까 그거 배울 때 참고 
```python
class Node:
  def __init__(self, data):
    self.data = data
    self.left = None
    self.right = None

class BinaryTree:
  def __init__(self):
    # 초기 트리 생성 시 root는 None
    self.root = None

  def insert(self, data):
    if not self.root:
      self.root = Node(data)
    else:
      # 루트가 있다면
      self._insert_recursive(self.root, data)

  def _insert_recursive(self, node: Node, data):
    # 값이 현재 노드보다 작을 경우
    if data < node.data:
      ## left가 없으면 바로 left만 채우기
      if not node.left:
        node.left = Node(data)
      ## left가 있으면 left로 재귀 호출
      else:
        self._insert_recursive(node.left, data)
    
    else:
      if not node.right:
        node.right = Node(data)
      else:
        self._insert_recursive(node.right, data)
```
#### 1. 전위 - pre-order (root, left, right)
```python
  def _preorder_traversal(self):
    result = []
    self._preorder_recursive(self.root, result)
    return result

  def _preorder_recursive(self, node: Node, result: list):
    if node:
      result.append(node.data)
      self._preorder_recursive(node.left, result)
      self._preorder_recursive(node.right, result)
```

#### 2. 중위 - in-order (left, root, right)
```python
  def inorder_traversal(self):
    result = []
    self._inorder_recursive(self.root, result)
    return result

  def _inorder_recursive(self, node: Node, result: list):
    if node:
      self._inorder_recursive(node.left, result)
      result.append(node.data)
      self._inorder_recursive(node.right, result)
```

#### 3. 후위 - post-order (left, right, root)

```python
  def _postorder_traversal(self):
    result = []
    self._postorder_recursive(self.root, result)
    return result

  def _postorder_recursive(self, node: Node, result: list):
    if node:
      self._postorder_recursive(node.left, result)
      self._postorder_recursive(node.right, result)
      result.append(node.data)
```


### 번외 : 트리 dict로 만들기 

참고 링크 : ~~~ 