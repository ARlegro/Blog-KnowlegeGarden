---
{"dg-publish":true,"permalink":"/DevStudy/Argorithm/DP, Greedy/개념/LCS(Longest Common Subsequence)/","noteIcon":"","created":"2025-12-03T14:52:51.816+09:00","updated":"2025-12-13T09:26:25.942+09:00"}
---




> 최장 공통 부분 수열 

DP문제 해결하는 방법 중 하나이다.

> [!WARNING] 부분 수열 vs 부분 문자열 
> ![Pasted image 20250801170811.png](/img/user/supporter/image/Pasted%20image%2020250801170811.png)
> 1. **부분 수열(Subsequence)** ✅
> 	- **원래 순서를 유지**한 채로 **일부 문자를 건너뛰어** 만든 수열 
> 	  
> 2. **부분 문자열(Substring)**
> 	- 연속되어야 한다.
ㅋ
> **LCS는 부분 수열을 사용**하는 알고리즘이다.
> 가장 긴 길이의 공통 부분 문자열을 찾는 문제 

---
## LCS란? 

> 두 수열에서 **순서는 유지**하되 **연속적이지 않아도** 되는 **가장 긴 부분 수열을 찾는 것** 

```Text
문자열 A: ACAYK P
문자열 B: CAP CAK
```
- 위의 문자열 A,B에서 공통된 부분 수열 중 제일 긴 것은 `ACAK`이다.

---
## 왜 DP에 LCS 적용해보기❓

### 1. 테이블 의미 파악 
LCS를 DP로 적용할 때는 **2차원 테이블**을 사용한다.
- 초기 테이블은 가로,세로 n+1개로 세팅
- `dp[i][j]`의 의미 : A문자열의 i번째, B문자열의 j번째까지의 LCS길이 
- i or j 가 0이면 비교할 문자가 없다는 뜻 

---
### 2. 경우에 따른 점화식 
크게 2가지 경우에 따라 다른 점화식을 도출할 수 있다.
1. `Str1[i-1] == Str2[j-1]`일 때 
	- `dp[i][j] = dp[i-1][j-1] + 1`	  
	<br>
2. `Str1[i-1] != Str2[j-1]`일 때 
	- `dp[i][j] = max(dp[i-1][j], dp[i][j-1])`
	- 이전 행 or 이전 열 중 더 긴쪽의 dp를 이어받는다.

---
### 3. 코드 맛보기 

> 뭘 말로 설명하냐. 그냥 코드 보면서 이해하는게 가장 빠름 
> 설명 참고: [LCS - velog글](https://velog.io/@emplam27/%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-%EA%B7%B8%EB%A6%BC%EC%9C%BC%EB%A1%9C-%EC%95%8C%EC%95%84%EB%B3%B4%EB%8A%94-LCS-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-Longest-Common-Substring%EC%99%80-Longest-Common-Subsequence)

#### 코드 
```python
import sys
input = sys.stdin.readline

def lcs_practice():
  A = "ABCDFF"
  B = "ACFVFSDF"

  # DP 테이블 생성 및 초기화
  ## dp[i][j] = A의 처음 i글자와 B의 처음 j글자까지의 LCS 길이
  n, m = len(A), len(B)
  dp = [[0]*(m+1) for _ in range(n+1)]
  
  # 앞쪽부터 ㄱㄱ
  for i in range(1, n + 1):
    for j in range(1, m + 1):
      if A[i-1] == B[j-1]:
        dp[i][j] = dp[i-1][j-1] + 1
      else: 
	      #문자가 다르면
        dp[i][j] = max(dp[i-1][j], dp[i][j-1])
      
      # 2개를 비교해서 max
      ## 1) dp[i-1][j]: 문자 A의 i-1까지와 문자 B의 j까지 비교한 결과
      ## 2) dp[i][j-1]: 문자 A의 i까지와 문자 B의 j-1까지 비교한 결과

  i, j = n, m
  reversed_lcs_chars = []
  # DP테이블의 오른쪽 아래 끝에서 시작하여 왼쪽 위 0,0까지 이동
  while i > 0 and j > 0:
    # 1. 현재 문자들이 같을 경우 (DP 세팅을 잘 했다면 대각선이랑 같다는 가정)
    if A[i-1] == B[j-1]:
      reversed_lcs_chars.append(A[i-1])

      i -= 1
      j -= 1
      continue
      
    # 2. 문자가 다르고 위쪽에서 현재 값에 도달했다면
    ## (str1의 문자를 포함하지 않은 경우)
    if dp[i-1][j] >= dp[i][j-1]:
      i -= 1

    # 3. 문자가 다르고, 왼쪽에서 현재 값에 도달한 경우
    ## (str2의 문자를 포함하지 않은 경우)
    else:
      j -= 1

  # 역순으로 저장되어 있으므로 뒤집어서 결과
  return "".join(reversed(reversed_lcs_chars))

print(lcs())
```

---
#### 코드 설명 1. DP 생성 부분 
```python
  for i in range(1, n + 1):
    for j in range(1, m + 1):
      if A[i-1] == B[j-1]:
        dp[i][j] = dp[i-1][j-1] + 1
      else: 
	      #문자가 다르면
        dp[i][j] = max(dp[i-1][j], dp[i][j-1])
```
- **같을 경우** : `dp[i][j] = dp[i-1][j-1] + 1`의 의미는❓
	- 두 문자가 같을 경우 지금까지의 LCS에 1을 더하는 간단한 로직 

- **다를 경우** : `max(dp(i-1, j), dp(j-1,i))`의 의미는❓
	- **부분 수열은 연속되지 않아도** 되기 때문에 생긴 로직이다.
	- **현재의 문자를 비교하는 과정 이전의 LCS는 계속해서 유지해야** 한다.
	- 이로 인해, 이 로직이 생긴 것 <br>![Pasted image 20250801173813.png](/img/user/supporter/image/Pasted%20image%2020250801173813.png)

---
#### 코드 설명 2. 역추적 부분 
> 1번만 해서는 최장길이는 구할 수 있지만, 최장길이에 해당하는 부분 수열을 구할 수 없다. 따라서, LCS부분 수열을 구하기 위해서는 역추적이라는 것이 필요

```PYTHON
  i, j = n, m
  reversed_lcs_chars = []
  # DP테이블의 오른쪽 아래 끝에서 시작하여 왼쪽 위 0,0까지 이동
  while i > 0 and j > 0:
    # 1. 현재 문자들이 같을 경우 (DP 세팅을 잘 했다면 대각선이랑 같다는 가정)
    if A[i-1] == B[j-1]:
      reversed_lcs_chars.append(A[i-1])
      i -= 1
      j -= 1
      continue
      
    # 2. 문자가 다르고 위쪽에서 현재 값에 도달했다면
    ## (str1의 문자를 포함하지 않은 경우)
    if dp[i-1][j] >= dp[i][j-1]:
      i -= 1

    # 3. 문자가 다르고, 왼쪽에서 현재 값에 도달한 경우
    ## (str2의 문자를 포함하지 않은 경우)
    else:
      j -= 1

  # 역순으로 저장되어 있으므로 뒤집어서 결과
  return "".join(reversed(reversed_lcs_chars))
```


>[!EXAMPLE] 로직 설명 
>1. **시작 지점** : DP배열의 마지막에서 시작한다 
>2. **세 가지 경우의 수 고려** 
>	1. **현재 문자가 LCS에 포함된 경우 (아래 조건 2개가 성립해야 함)**
>		- 조건 1.  `Str1[i-1] == Str2[j-1]`
>		- 조건 2. `DP[i][j] == DP[i-1][j-1] + 1`
>		- 이 때 현재 문자열은 LCS안에 들어간다.
>		- 이동 : 왼쪽 대각선 방향 Cuz 지금 문자를 제외한 이전 상태로 돌아가기 위해 
>	2. **Str1의 문자를 버린 경우**
>		- 조건 1. `Str1[i-1] != Str2[j-1]`
>		- 조건 2. `DP[i][j] == DP[i-1][j]
>		- 의미 : 현재 `dp[i][j]`는 `dp[i-1][j]`에서 내려온 것이다. 즉, `Str1[i-1]`문자는 포함시키지 않고 `dp[i-1][j]`로 이동 
>		  
>	3. **Str2의 문자를 버린 경우**
>		- 조건 1. `Str1[i-1] != Str2[j-1]`
>		- 조건 2. `DP[i][j] == DP[i][j-1]
>		- 의미 : 현재 `dp[i][j]`값은 `dp[i][j-1]`에서 내려온 값이다. 즉, Str2의 j-1번까지의 LCS값인 `dp[i][j-1]`이 현재 `dp[i][j]`값과 같다는 의미이므로 Str2의 현재 문자열을 버리고 `dp[i][j-1]`로 이동한다.
