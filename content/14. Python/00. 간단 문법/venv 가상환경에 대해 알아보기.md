---
tags:
  - venv
---
# Python 가상환경 .venv 란?

## 한 줄 요약
- 프로젝트별로 독립된 Python 패키지 공간을 만드는 도구.
---
## 왜 필요한가?
- Python은 패키지를 전역으로 설치하기 때문에 프로젝트마다 버전 충돌이 생긴다.
- 아래와 같은 문제 상황이 발생할 수 있다.
```
프로젝트 A → requests 2.28 
프로젝트 B → requests 2.31 ← 충돌!
```
- `.venv`를 사용하면 프로젝트마다 패키지를 격리해서 관리할 수 있다.
---
## 사용 순서
- 활성화하면 터미널 프롬프트 앞에 `(.venv)`가 붙는다.
```bash
# 1) 가상환경 생성
python -m venv .venv

# 2) 가상환경 활성화
source .venv/bin/activate    # Mac/Linux
.venv\Scripts\activate       # Windows

# 3) 패키지 설치 (.venv 안에만 설치됨)
pip install -r requirements.txt

# 4) 비활성화
deactivate

# 5) 설치 된 venv 삭제
rm -rf .venv
```
