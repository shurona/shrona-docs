---
tags:
  - KMP
---

# 기본 접근
아래의 표에서 기본 접근을 확인할 수 있다. 

| .   | a   | b   | a   | a   | b   | c   | .   | .   |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
|     |     |     |     |     |     | ↑   |     |     |
|     | a   | b   | a   | a   | b   | a   |     |     |
|     |     |     |     | a   | b   | a   | a   | b   |

KMP는 접두사와 접미사의 개념을 활용하여서 모든 경우의 수를 계산하지 않고 반복되는 연산을 줄여나가는 기법이다.
이를 위해서는 접두사와 접미사를 이용해서 pattern이 얼마나 겹치는 지 전처리 작업이 필요하다.

###  실패 함수(Failure Function)
- 실패 함수는 KMP 알고리즘에서 사용하는 보조 데이터 구조로, 문자열 패턴 내에서 접두사와 접미사가 일치하는 최대 길이를 저장합니다.
- 패턴 매칭 중 일치 실패가 발생하면, 실패 함수 값을 이용하여 비교 위치를 효율적으로 이동합니다.
- 이를 통해 이미 일치한 부분 문자열의 정보를 활용하고, 불필요한 중복 비교를 방지합니다.
### 예시 
만약 문자열 abacaaba가 있다고 가정할 때 아래와 같이 접두사와 접미사가 동일한 최대 문자열을 찾아야 한다.

| **i** | **pattern(0...i)** | **접두사이면서 접미사인 최대 문자열** | **table[i]** |
| ----- | ------------------ | ---------------------- | ------------ |
| 0     | a                  |                        | 0            |
| 1     | ab                 |                        | 0            |
| 2     | aba                | a                      | 1            |
| 3     | abac               |                        | 0            |
| 4     | abaca              | a                      | 1            |
| 5     | abacaa             | a                      | 1            |
| 6     | abacaab            | ab                     | 2            |
| 7     | abacaaba           | aba                    | 3            |
위의 예시로 보면 pi[] 테이블을 아래와 같이 표현할 수 있다.
```Java
int[] pi = [0, 0, 1, 0, 1, 1, 2, 3]
```

## 코드로 접근
먼저 전처리 PI 테이블을 아래와 같이 작성할 수 있다.
```Java
public int[] calPi(String find) {  
  
        int[] pi = new int[find.length()];  
  
        int left = 0;  
        for (int right = 1; right < find.length(); right++) {  
            // 스텝이 올라갔을 때 다른 게 나오면 위치를 조정해 준다. 
            while (left > 0 && find.charAt(left) != find.charAt(right)) {  
                left = pi[left - 1];  
            }  
  
            // 만약 두 개의 값이 같으면 left를 하나 올려준다.  
            if (find.charAt(left) == find.charAt(right)) {  
                left += 1;  
            }  
  
            // 얼마나 중복이 되었는지 저장해준다.
            pi[right] = left;  
  
        }  
  
        return pi;  
    }
```

#### KMP 로직
앞에서 부터 하나씩 비교해 나가면서 중복된 접미사와 접두사의 경우 위치 포인트를 옮겨준다.
```Java
int count = 0;  
int pointer = 0;  
for (int left = 0; left < origin.length(); left++) {  
    // 같으면 뭐를 해야 할까  
    while (pointer > 0 && origin.charAt(left) != find.charAt(pointer)) {  
        pointer = pi[pointer - 1];  
    }  

	// 길이가 같으면 비교하는 포인터의 위치를 하나씩 옮겨준다.
    if (origin.charAt(left) == find.charAt(pointer)) {  
        pointer += 1;  
    }  
  
    // 문자열이 일치 한 경우 로직을 처리 한다.
    if (pointer == find.length()) {  
        count += 1;  
        // 현 시점에서 접두사와 접미사의 동일성을 조정해서 포인터 위치를 지정해준다.
        pointer = pi[pointer - 1];  
    }  
}
```

## 패턴의 최소 반복 주기
```Java
pattern.length - pa[pattern.length - 1];
```
`pa[pattern.length -1]` pi 배열(또는 실패 함수)의 마지막 값으로, 패턴의 마지막 문자까지 고려했을 때 접두사와 접미사가 일치하는 최대 길이를 의미합니다.
##### 의미 
- 패턴의 가장 긴 접두사-접미사의 길이입니다.
- 이 값을 전체 길이에서 빼면, 패턴이 반복되지 않은 부분(즉, 최초의 주기)의 길이를 구할 수 있습니다.
- 주기가 되는 이 값은 패턴이 얼마나 짧은 단위로 반복되는지를 알려줍니다.

패턴에서 나머지 부분(pattern.length - pa[pattern.length - 1])이 반복 단위로 쓰일 수 있는 이유
1. 접두사와 접미사가 일치한다면, 그 사이의 나머지 부분이 패턴의 최소 단위로 반복될 수 있다.
2. 이 값이 최소 단위의 길이를 나타내며, 패턴이 전체 길이에서 반복되는 방식으로 나타난다는 것을 보장합니다.
### 예시
#### 패턴
- aabaaa
길이
- 6
pi  배열
- [0, 1, 0, 1, 2, 2]
- pi[5] = 2  (가장 긴 접두사-접미사의 길이)

계산
pattern.length - pa[pattern.length - 1]
= 6 - 4 = 2

#### 의미
- 이 값 4는 패턴의 최소 반복 단위의 길이를 나타냅니다.
- 즉, aabaaa는 aaba가 최소 단위로 반복되며 생성된 패턴입니다.
