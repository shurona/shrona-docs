---
tags:
  - ssh
---
# ssh 키란
SSH(Secure Shell)는 네트워크 상에서 안전하게 원격 접속을 할 수 있도록 해주는 암호화 기반 프로토콜입니다. 비밀번호 대신 공개키/개인키 방식의 인증을 사용하면 보안성과 편의성을 모두 얻을 수 있습니다.
# ssh 키 생성하기
## 키 생성 명령어
```bash
ssh-keygen -t rsa -b 4096
```
- `-t`: 키 타입 (보통 rsa 또는 ed25519)
- `-b`: 키 길이 (rsa의 경우 4096 추천)
## 과정
- 명령어 실행 후, 키를 저장할 경로와 파일명을 입력합니다. (기본값: `~/.ssh/id_rsa`)
- 키 사용 시 필요한 비밀번호(passphrase)를 입력합니다. (비워두면 무암호)
- 위 과정을 마치면 **개인키**(id_rsa)와 **공개키**(id_rsa.pub)가 생성
## 권한 설정
```bash
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
```
- 개인키는 반드시 600권한이어야 한다.
## EC2 연결
ec2의 `~/.ssh/authorized_keys` 파일에 추가 하거나 공개키 파일을 ssh 폴더 내부에 저장을 한다.
이후 `ssh ${host}@${url}`fh wjqrmsgkf tn dlTek.
# ssh config로 키 관리 자동화
```bash
# ~/.ssh/config
Host myserver
  HostName 1.2.3.4
  User ubuntu
  Port 2222
  PreferredAuthentications publickey
  IdentityFile ~/.ssh/my_private_key
  ServerAliveInterval 60
```
- 위와 같이 config로 등록해놓으면 `ssh myserver`로 바로 접속이 가능하다.
- `PreferredAuthentications publickey`를 사용하면 ssh키를 우선적으로 확인해서 접속한다.
# ssh를 활용한 접근
## 포트포워딩
원격 서버의 특정 포트를 로컬 포트와 연결할 수 있다.
```bash
ssh -L [로컬포트]:[원격주소]:[원격포트] myserver
```
- 예시
	- `ssh -L 8080:localhost:80 myserver`
## SCP로 파일 전송
- 원격 서버로 파일 전송
```bash
scp -P 2222 -i ~/.ssh/my_private_key localfile.txt user@server:/remote/path/
```
- 원격 서버에서 파일 받아오기
```bash
scp -P 2222 -i ~/.ssh/my_private_key user@server:/remote/path/file.txt ./localdir/
```
- `-P` 포트 지정
- `-i` 개인키 지정
- `-r` 폴더 전체 전송