---
{"dg-publish":true,"permalink":"/DevStudy/Argorithm/DP, Greedy/개념/Unbounded, Fractional Knapsack/","noteIcon":"","created":"2025-12-03T14:52:51.828+09:00","updated":"2025-12-13T18:25:27.233+09:00"}
---



이전 : [[DevStudy/Argorithm/DP, Greedy/개념/Knapsack Problem 개념 + 0-1 Knapsack\|Knapsack Problem 개념 + 0-1 Knapsack]]

## 종류2. Unbounded Knapsack
#제한없는 #여러번 

### 개념 
- **각 물건을 원하는 만큼 여러 번** 담을 수 있다.
- 동전 문제가 이런 것 

>같은 물건을 여러 번 선택하여 배낭에 담을 수 있다.
>- **2차원 배열 방식**은 0-1 냅색이랑 코드가 거의 같다.
>- **1차원 배열 방식**이 다른데, 이걸 많이 쓰니 잘 알아볼 것 

---
### 2차원 배열 
```python title:완전냅색_2D
def unbounded_knapsack_2d(W, N, items:list):

  '''
  제한없는냅색 문제를 2차원 DP로 해결하는 함수
  (각 물건은 여러 번 선택이 가능)
  Args:
    W : 배낭의 최대 용량
    N : 물건의 개수
    items : 각 물건의 튜플리스트 ((무게, 가격))
  
  Returns:
    배낭에 담을 수 있는 최대 가치
  '''
  # dp[i][w] = i번째 물건까지 고려했을 때, 배낭 용량 w에서의 최대 가치
  dp = [[0] *(W+1) for _ in range(N+1)]

  # 물건들을 하나씩 순회하면 DP 테이블 채우기
  ## i = 현재 고려하는 물건의 인덱스
  for i in range(1, N+1):
    weight, value = items[i-1]
    # w는 현재 배낭의 용량
    for w in range(1, W+1):
      if weight > w:
        dp[i][w] = dp[i-1][w]
      else:
        dp[i][w] = max(dp[i-1][w], dp[i][w - weight] + value)
```

이건 0-1 Knapsack_2D 방식이랑 같으니 설명 생략 

---
### 1차원 배열 사용 
> 공간 효율성 때문에 많이 사용 ⭐

```python title:unbounded_knapsack_1d
def unbounded_knapsack_1d(W, N, items:list):

  '''
  제한없는냅색 문제를 1차원 DP로 해결하는 함수
  (각 물건은 여러 번 선택이 가능)
  '''

  # dp[i][w] = i번째 물건까지 고려했을 때, 배낭 용량 w에서의 최대 가치
  dp = [0 for _ in range(W+1)]

  # 물건들을 하나씩 순회하면 DP 테이블 채우기
  ## i = 현재 고려하는 물건의 인덱스
  for i in range(1, N+1):
    weight, value = items[i-1]

    # 핵심 : 배낭 용량을 순 방향으로 순회 unlike 0-1 1d
    ## 이렇게 해야 현재 물건 i 를 여러 번 사용하는 효과를 낼 수 있다.
    ## weight부터 시작 : 어차피 그 이전은 현재 물건의 weigh로는 변화를 줄 수 없느니 그대로(1차원이라 가능)
    for w in range(weight, W+1):
      dp[w] = max(dp[w], dp[w-weight] + value)

  return dp[w]


w = 10
N = 3
items = [(2, 3), (3, 4), (5, 7)]

max_value = unbounded_knapsack_1d(w, N, items)
print(f"제한 없는 냅색을 사용한 최대 가치: {max_value}") # 15
```
- 전체적인 로직은 여태까지의 Knapsack과 거의 흡사하다.
- 단지, 1차원 배열임에도 **순방향으로 순회해야 한다**는 점만 다르다(**여러 번 사용하는 효과**를 내기 위해)


---

## 종류3. Fractional Knapsack
#분할 #Greedy 

### 개념 
> Greedy를 사용해서 해결하는 냅색 

- **물건을 쪼개서 담을 수 있다.**(일부만)
	- 이러한 특성 때문에 Greedy 방법이 빛을 냄 Cuz 일단 가치 있는 물건을 담고 공간이 부족하면 **물건을 쪼개서라도 최대한 담는 것이 항상 최적**이기 때문 
	  
>[!QUESTION] Greedy로 풀리는 이유 
>- **이전 Knapsack이 Greedy가 힘들었던 이유** 
>	- '0-1' or '제한 없는' Knapsack은 물건을 쪼갤 수 없었다 ➡ 나중에 더 좋은 선택지가 나올 수 있기에 어떤 물건을 선택할지 말지를 정하기가 어려웠다 
>- **'Fractional' Knapsack은** 
>	- **쪼갤 수 있기** 때문에 가장 **가치가 높은 물건부터 채워 넣는 것이 항상 최적의 전략**이 된다
>	- 단위 무게당 가치 = 총가치 / 총 무게   << 이렇게 하면 각 물건의 단위 가치 비교 가능

---
### 해결 과정 이론 
1. **단위 무게당 가치를 계산**한다 
2. **정렬** : 단위 무게당 가치가 높은 순서대로 물건들을 정렬 
3. **배낭 채우기** : 정렬된 순서대로 물건을 배낭에 채워 넣기 

---
### 코드 
```python 
def fractional_knapsack(W, N, items:list):
  '''
  Fractinal Knapsack 문제를 Greedy로 해결하는 함수 
  물건을 쪼개는 것이 Key-Point

  Args:
    W : 배낭의 최대 무게 용량
    N : 물건의 개수 
    items : 각 물건의 튜플리스트 - (무게, 가치)

  Returns:
    float : 배낭에 담을 수 있는 최대 가치
    (쪼갤 수 있으므로 float)
  '''

  # 1. 각 물건의 튜플리스트 생성 - (단위 무게당 가치, 무게, 가치)
  item_ratios = []
  for w, v in items:
    # 무게가 0인 경우 방지
    if w == 0:
      item_ratios.append((float('inf'), w, v))
    else:
      item_ratios.append(((v/w), w, v))
  
  # 2. 단위 무게당 가치가 높은 순서대로 정렬 (내림차순)
  ## 튜플의 첫 요소인 '단위 무게당 가치'를 기준으로 정렬 
  item_ratios.sort(key=lambda x:x[0], reverse=True)

  current_weight = 0  # 현재 배낭에 담긴 총 무게 
  total_value = 0.0  # 현재 배낭에 담긴 총 가치 

  # 3. 정렬된 순서대로 물건을 배낭에 채우기 
  for ratio, weight, value in item_ratios:
    ## 현재 물건 '전체'를 배낭에 담을 수 있는 경우 
    if current_weight + weight <= W:
      current_weight += weight
      total_value += value
    else:
      # 쪼개서 담을 용량 
      remaining_capacity = W - current_weight

      # 쪼개서 만든 가치 
      total_value += ratio * remaining_capacity
      current_weight += W  # 배낭이 꽉 찼으므로 current_w를 최대용량으로
      break # 배낭이 꽉 찼으므로 이제 종료 

  return total_value


W = 50
N = 3 
items = [(60, 100), (100, 120),(30, 60)]

max_gold = fractional_knapsack(W, N, items)
print(f'Fractional Knapsack을 사용한 최대 금 양 = {max_gold}')
```
- **복잡도**
	- **시간 복잡도** : $O(NlogN)$ 
		- 순회 : O(N)
		- **정렬** : $O(NlogN)$ 
	- **공간 복잡도** : O(N)
		- 물건 정보들 저장하는데 O(N)공간 사용 
