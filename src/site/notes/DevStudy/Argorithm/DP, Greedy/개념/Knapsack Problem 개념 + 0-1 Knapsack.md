---
{"dg-publish":true,"permalink":"/DevStudy/Argorithm/DP, Greedy/개념/Knapsack Problem 개념 + 0-1 Knapsack/","noteIcon":"","created":"2025-12-03T14:52:51.850+09:00","updated":"2025-12-13T18:25:27.218+09:00"}
---

Next : [[DevStudy/Argorithm/DP, Greedy/개념/Unbounded, Fractional Knapsack\|Unbounded, Fractional Knapsack]]



>[!tip] 혼틈 DP 꿀팁
>- Problem을 SubProblem으로 쪼갤 수 있어야 함 
>- 근데 잘못 쪼개면 안됨 

---
## 개념 
- Knapsack = 배낭.
- Knapsack Problem = **제한된 배낭 용량 안에 물건들을 담아 최대 가치를 얻는 방법을 찾는** 문제 

> Knapsack은 크게 3가지 유형으로 나뉜다.
> - 0-1 냅색 
> - 제한 없는 냅색
> - 분할 냅색 

---
## 종류 1. '0-1 Knapsack'
### 개념 
#모or도
- 각 물건을 **넣거나(1) 넣지 않거나(0) 둘 중 하나만 선택**할 수 있다.
- ❌한 번 선택한 물건은 다시 선택 불가 
- ❌물건을 쪼개서 담을 수 없다 
- **단순히** True, False로만 생각하면 **경우의 수는 $2^n$** 이다. 근데 이렇게 되면 n이 10만 되어도 1024가지의 경우가 되어버리니 **최적화가 필요**.(DP로 푸는데 이 부분은 뒤에서)
- **복잡도** (이건 나중에 코드 보면 이해됨)
	- 시간 복잡도 : $O(N*W)$  ⬅ N개의 물건을 순회하면서 W개의 용량을 순회하므로 
	- 공간 복잡도 : $O(N*W)$  

---
### 장단점 
- **✅장점**
	- **구현이 간단** : 2차원 배열이 직관적 (1차원 배열로도 가능함)
	  
- **💢단점**
	- 물건의 개수 or 배낭의 최대 무게가 클 경우 **메모리 부담**  ⬅ 이때는 **1차원 배열로 최적화 必**

---
### 코드 
#### 2차원 배열 사용 
```python title:0-1_Knapsack-2d
def knapsack_02(W, N, items: list):

  '''
  0-1 Knapsack Problem을 2차원 DP로 해결하여 최적화
  Args:
    W : 배낭의 최대 무게 용량
    N : 물건의 개수
    items : 각 물건의 튜플리스트((무게, 가치))
  '''

  # dp[i][w] = i번째 물건까지 고려했을 때!!, 배낭 용량 w에서의 최대 가치
  dp = [[0] *(W+1) for _ in range(N+1)]
  for i in range(1, N+1):
    weight, value = items[i-1]
    # w = 현재 배낭의 용량
    for w in range(1, W+1):
      # 1. 현재 물건(i)을 배낭에 넣을 수 없는 경우
      ## i-1번째 물건까지 고려했을 때와 동일한 가치
      if weight > w:
        dp[i][w] = dp[i-1][w]

      # 2. 넣을 수 있는 경우
      ## a) i번째 물건을 넣지는 않은 경우
      ## b) i번째 물건을 넣은 경우
      else:
        dp[i][w] = max(dp[i-1][w], dp[i-1][w - weight] + value)

  # dp[N][M] : N개의 물건을 모두 고려했을 때, 배낭 용량이 W일 때의 최대 가치
  return dp[N][W]

###
W = 7
N = 4
items = [(6, 13), (4, 8), (3, 6), (5, 12)]
max_value = knapsack_01(W, N, items)
print(f'2차원 DP 배열 사용 최대 가치 = {max_value}')
```
1. **DP 배열 생성** : i번째까지 물건을 고려했을 때, 특정 배낭 용량(w)에서의 최대가치를 담을
2. **배낭 용량별 i번째 item을 넣을 수 있는 경우 없는 경우를 나눈다**
	- **넣을 수 없는 경우** : i-1번째 경우의 최대 가치와 동일하게 
	- **넣을 수 있는 경우** : i번째 물건을 넣을 경우/넣지 않을 경우 중 max값으로 
3. **최종 출력** : N개의 물건을 고려하고 배낭 용량이 w일 때의 최대 가치 
---
#### 1차원 배열 사용 
```python title:0-1_Knapsack_1d
def knapsack_01_1d(W, N, items:list):
    """
    0-1 Knapsack Problem을 1차원 DP 배열로 공간을 최적화하여 해결하는 함수.
    """
    # dp[w] = 현재까지 고려한 물건들로 배낭 용량 w를 채웠을 때의 최대 가치
    dp = [0] * (W + 1)

    # 기존 방식처럼 2중 FOR문
    ## 물건들을 하나씩 순회하면 DP 배열 업데이트
    ## i = 현재 고려하는 물건의 인덱스
    for i in range(N):
        weight, value = items[i]
        # 중요 : 배낭 용량을 역순으로 순회!!! Cuz 0-1 Knapsack
        for w in range(W, weight - 1, -1):
            dp[w] = max(dp[w], dp[w-weight] + value)

    print(dp) # [0, 0, 0, 6, 8, 12, 13, 14]
    return dp[W]

W = 7
N = 4
items = [(6, 13), (4, 8), (3, 6), (5, 12)]
max_value = knapsack_01_1d(W, N, items)
print(f'1차원 DP 배열 사용 최대 가치 = {max_value}') # 14
```


1. **DP 배열의 의미**
	- `dp[w]` : 지금까지 고려한 물건들 중 배낭 용량이 정확히 W일 때 얻을 수 있는 최대 가치 

2. **역방향으로 순회하기** ⭐
	- ❌만약 순방향으로 순회하면 한 물건을 여러번 참조하게 되고, 이는 한 물건을 여러 번 담는 "제한 없는 냅색"이 되어버린다. 
	- 01_Knapsack은 **한 물건을 두 번 이상 담으면 안됨으로 역방향 순회 必**![Pasted image 20250802002900.png](/img/user/supporter/image/Pasted%20image%2020250802002900.png)
	- 위의 사진을 참고하고  `dp[w] = max(dp[w], dp[w - weight_i] + value_i)` 코드를 보면, 특정 w값의 최대가치를 계산할 때 이전 인덱스의 값인 `dp[w-weight]`를 참고하는 것을 볼 수 있다. 💢**만약 순 방향으로 순회**를 했다면 **이미 해당 item을 반영한 경우의 최대가치를 참조**하는 것이므로 **여러 번 담기는 경우의 최대가치로 계산해버릴 수 있다.** **✅따라서 역방향**으로 ㄱ 

> vs 2차원 배열 사용
> - **시간복잡도는 똑같다** : 여전히 N개의 물건과 W개의 용량 순회 By 이중 for문 
> - **공간 복잡도는 `/N`만큼 개선** : $O(W)$


