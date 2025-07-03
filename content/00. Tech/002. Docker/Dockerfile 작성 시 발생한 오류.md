---
tags:
  - dockerfile
  - alpine
---
# RDS를 dump 하려고 할 때 발생한 에러
## Dockerfile을 작성한 목적
1. postgres:13.18 기반 컨테이너 이미지 빌드
2. SSH 터널을 통해 원격 DB에 접속
3. pg_dump 로 데이터 덤프 생성
이를 위해 Dockerfile 에 openssh-client 를 설치하고, CI 스크립트에서 ENTRYPOINT 단계에 바로 SSH-pg_dump 스크립트를 실행하도록 구성했습니다.
### 기존 문제 Dockerfile
```Dockerfile
FROM postgres:13.18
USER root

# SSH 설치
RUN apt-get update && \
    apt-get install -y openssh-server
# 사용자 전환
USER postgres
```
## 발생한 문제 1
```text
E: Problem executing scripts APT::Update::Post-Invoke 'rm -f /var/cache/apt/archives/*.deb …'
E: Sub-process returned an error code
The command '/bin/sh -c apt-get update && apt-get install -y openssh-client && rm -rf /var/lib/apt/lists/*' returned a non-zero code: 100
```
### 원인
Debian/Ubuntu 계열 기본 이미지에는 docker-clean 이라는 APT 후처리 훅이 들어 있다.   
apt-get update 직후에 아래 스크립트를 실행하도록 설정되어 있다.   
하지만 이 과정에서 해당 폴더가 없거나 권한 문제 등으로 
### 해결방법
해당 훅을 삭제하면 패키지 설치가 가능하다.
```Dockerfile
FROM postgres:13.18
USER root

# 1) docker-clean 훅 삭제
RUN rm -f /etc/apt/apt.conf.d/docker-clean

# 2) 안전하게 APT 패키지 설치
RUN apt-get update \
 && apt-get install -y --no-install-recommends openssh-client \
 && rm -rf /var/lib/apt/lists/*

USER postgres
```
## 발생한 문제 
- Cannot allocate memory lzma 압축 해제 오류
### 원인
- Gitlab에서 CI를 돌리는 상황에서 아래와 같은 에러가 발생하였다.
```text
lzma error: Cannot allocate memory
tar: This does not look like a tar archive
dpkg-deb: subprocess returned error exit status 2
Errors were encountered while processing:
 /var/cache/apt/archives/libcbor0.8_0.8.0-2+b1_amd64.deb
 … (이하 생략)
E: Sub-process /usr/bin/dpkg returned an error code (1)
```
.deb 패키지 내부의 control.tar.xz 압축 해제 시에 LZMA 라이브러리가 많은 메모리를 요구하는데, CI 동작 노드(특히 DinD 컨테이너) 쪽에 할당된 메모리가 부족해 터지는 문제였습니다.
### 해결 방법
- Alpine 기반으로 전환한다.
```Dockerfile
FROM postgres:13.18-alpine

USER root
RUN apk update \
	&& apk add --no-cache openssh-client

USER postgres
```
- 메모리 소모가 적어 CI 환경에서 안정적이다.
- 이미지의 용량이 감소한다.
- Alpine 패키지 설치 속도도 빠르다.
## 결론
- **Debian → Alpine 전환** 은 메모리·용량·빌드 속도 측면에서 유리하다.
- Alpine 이미지는 APT(Debian/Ubuntu 계열)가 아니라 자체 패키지 매니저인 apk 를 쓰기 때문에 애초에 그 “APT::Update::Post-Invoke” 훅 자체가 없다.