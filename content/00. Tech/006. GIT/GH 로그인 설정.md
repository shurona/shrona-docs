---
tags:
  - GH
---

# GH 로그인 설정

## SSH 인증과 `gh` 인증의 차이

- 기존 SSH 키는 `git push`, `git pull`처럼 Git 저장소에 접근할 때 사용한다.
- `gh pr create`처럼 GitHub API를 호출하는 기능은 GitHub CLI의 OAuth 인증이 별도로 필요하다.
- `gh auth login` 중 표시되는 `Upload`는 로컬 SSH 공개키를 GitHub에 새로 등록할지 묻는 단계다.
- 기존 SSH 키로 `git push`가 정상 동작한다면 키를 다시 올릴 필요가 없다. 해당 화면에서는 `Skip`을 선택한다.

## 기존 SSH 키를 유지한 채 로그인하기

```bash
gh auth login -h github.com -p ssh --skip-ssh-key --web
```

각 옵션의 의미는 다음과 같다.

- `-h github.com`: 인증할 GitHub 호스트
- `-p ssh`: Git 작업에 SSH 프로토콜 사용
- `--skip-ssh-key`: 기존 SSH 키를 다시 업로드하지 않음
- `--web`: 브라우저에서 GitHub CLI용 OAuth 인증 진행

브라우저 인증을 완료하면 기존 SSH 키는 그대로 유지하면서 `gh`의 PR 생성 등 API 기능을 사용할 수 있다.

## 인증 확인

```bash
gh auth status
git remote -v
ssh -T git@github.com
```

- `gh auth status`: GitHub CLI 로그인 상태 확인
- `git remote -v`: 저장소 remote가 SSH 주소인지 확인
- `ssh -T git@github.com`: 등록된 SSH 키로 GitHub 인증이 되는지 확인

`ssh -T` 실행 시 GitHub가 셸 접근을 제공하지 않는다는 문구와 함께 사용자명이 표시되면 SSH 인증은 정상이다.
