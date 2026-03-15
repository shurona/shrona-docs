---
tags:
  - python/grammar
---
# 페이지 설명
- 파이썬으로 코딩 테스트 문제 풀면서 적을만한 문법 정리
- ex> Java Stream

## f-String으로 출력하기
```python
name = "Iron"
level = 90

print(f"캐릭터: {name}, 레벨: {level}")

# 출력: 캐릭터: Iron, 레벨: 90
```
## 문자열  처리
- 문자열 길이
	- `len(s)`
- 문자열 파싱
``` python
s = "3232323"

s[0:3]   # "323"  — 0번째부터 3번째 전까지
s[2:5]   # "3232"[2:5] = "323"
s[1:]    # "232323" — 1번째부터 끝까지
s[:4]    # "3232"  — 처음부터 4번째 전까지
s[-3:]   # "323"   — 뒤에서 3개
```
## 반복문 처리
### 기본 range
```python
for i in range(5):        # 0, 1, 2, 3, 4
for i in range(1, 6):     # 1, 2, 3, 4, 5
for i in range(0, 10, 2): # 0, 2, 4, 6, 8 (step)
for i in range(5, 0, -1): # 5, 4, 3, 2, 1 (역순)
```
### 리스트 순회
```python
arr = [10, 20, 30]
for x in arr:             # 10, 20, 30
```
### 인덱스 + 값 동시에 (Java의 일반 for랑 동일)
```python
for i, x in enumerate(arr):  # (0, 10), (1, 20), (2, 30)
```
### 2차원 배열
```python
matrix = [[1,2],[3,4],[5,6]]
for row in matrix:
    for val in row:
        print(val)
```
### 문자열 순회
```python
for ch in "hello":        # h, e, l, l, o
```
### zip — 두 리스트 동시 순회
```python
a = [1, 2, 3]
b = ["a", "b", "c"]
for x, y in zip(a, b):   # (1,"a"), (2,"b"), (3,"c")
```
### 딕셔너리
```python
d = {"a": 1, "b": 2}
for key in d:             # key만
for key, val in d.items() # key + value 둘 다
```
