---
{"dg-publish":true,"permalink":"/DevStudy/Argorithm/Merge Sort/","noteIcon":"","created":"2025-12-03T14:52:51.952+09:00","updated":"2025-12-09T17:19:42.635+09:00"}
---


### 개념 

- 배열을 절반씩(앞부분과 뒷부분) 나누어 각각 정렬한 후 두 정렬된 배열을 병합하는 작업을 반복하는 알고리즘 
- 배열의 원소 수가 2개 이상인 경우의 흐름
	1. 배열의 **앞부분을 병합 정렬**로 정렬
	2. 배열의 **뒷부분을 병합 정렬**로 정렬 
	3. 배열의 앞/뒷 부분을 **병합** 
- 별도의 공간을 필요로 함 ➡ 공간이 부족하면 대신 퀵 소트를 이용

>[!example] 시간복잡도 
>- N개 쪼개져야하고, 한번 호출당 절반씩 줄어드니 logn ➡ N logn
>- 퀵소트도 최악만 아니면 n logn임
![Pasted image 20250717204353.png](/img/user/supporter/image/Pasted%20image%2020250717204353.png)

>[!tip] 모듈 heapq -> merge()함수 


함수가 호출될때마다 1/2씩 잘라서 재귀적으로 호출하고 맨끝부터 2개씩 병합해서 정렬된 배열을 머지해나가는 방법

### 머지 소트 구현해보기 

#### java 
```java
// 1. 준비  
private static void merge(int[] arr){  
    // 임시 배열 생성 : 정렬할 배열의 크기만큼  
    int[] tmp = new int[arr.length];  
    // 파라미터 : 정렬할 배열/ 임시저장소/ 시작 인덱스/끝 인덱스  
    mergeSort(arr, tmp, 0, arr.length - 1);  
}  
  
// 2. 머지 시작 - 재귀
private static void mergeSort(int[] arr, int[] temp, int start, int end){  
    if (start < end){  
        // 함수 호출 시 2개로 자르는 작업 必  
        int mid = (start + end) / 2;  
        mergeSort(arr, temp, start, mid);  
        mergeSort(arr, temp, mid+1, end);  
        // 이제 2개의 부분이 정렬되어있는 상태  
        // 3. merge() : 2개의 배열을 병합하는 함수  
        // 기존 mergeSort 파라미터 + mid 인덱스  
        merge(arr, temp, start, mid, end);  
  
    }  
}  

// 3. 소트한것들 합치기 
private static void merge(int[] arr, int[] temp, int start, int mid, int end){  
    // 임시 저장소에 정렬이 된 배열을 필요한 만큼 복사 ✅ 
    for (int i = start; i <= end; i++) {  
        temp[i] = arr[i];  
    }  
    // 일단 이전에 나눴던 두 배열의 start 부분 index 구하기  
    // 비교하면서 포인트 옮길 것  
    int part1 = start;  
    int part2 = mid + 1;  
    // 현재 포이터가 배열에 어디에 위치하는지도 알아야 하므로 for 저장
    int index = start;  
    // 두 그룹 비교 하면서 작은 것 먼저 넣기 
    while (part1 <= mid && part2 <= end){  
        if (temp[part1] <= temp[part2]){  
            arr[index] = temp[part1];  
            part1++;  
        } else {  
            arr[index] = temp[part2];  
            part2++;  
        }  
        index++;  
    }  
    // 위의 while은 두 배열 중 하나만 끝났을 때 다  
    // 따라서 안 끝난 배열도 넣어줘야 함  
    // 근데 뒤쪽배열이 남아있다면 이미 뒤쪽이기에 처리할 필요 없다  
    for (int i = 0; i <= mid-part1; i++){  
        arr[index + i] = temp[part1 + i];  
    }  
}  
  
public static void main(String[] args) {  
    int[] arr = {3,9,4,7,5,0,1,6,8,2};  
  
    printArray(arr);  
    merge(arr);  
    printArray(arr);  
    
    // 3, 9, 4, 7, 5, 0, 1, 6, 8, 2, 
		// 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 
}  
  
public static void printArray(int[] arr){  
    for (int i : arr) {  
        System.out.print(i + ", ");  
    }  
    System.out.println();  
}
```

**temp 이유** 
- 각 재귀 호출마다 비교를 해야하는데 이 temp를 두 부분으로 나눠서 비교할 것 
![Pasted image 20250717220613.png](/img/user/supporter/image/Pasted%20image%2020250717220613.png)
- 사진처럼 복사본을 만들어서 ➡ 그 복사본을 두 부분으로 만들고 ➡ 각각을 비교하면서 순서대로 원본에 해당 인덱스 범위까지 대입 
