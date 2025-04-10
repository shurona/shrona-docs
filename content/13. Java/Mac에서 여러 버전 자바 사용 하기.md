---
tags:
  - java/version
---
# Homebrew와 `/usr/libexec/java_home` 활용

**Homebrew**를 사용하면 쉽게 여러 JDK 배포판을 설치할 수 있습니다. 예를 들어, Temurin 기반의 배포판이 아닌 OpenJDK를 사용하고자 한다면, Homebrew Cask를 활용할 수 있습니다.

### 설치

터미널에서 아래 명령어로 Java 17과 Java 21을 설치합니다:

```bash
brew install --cask openjdk@17
brew install --cask temurin@21
```

설치가 완료되면, `/usr/libexec/java_home` 명령어를 사용하여 원하는 버전의 JAVA_HOME을 설정할 수 있습니다.
## Java_Home 설정 방법
## java 17 사용하기
```shell
export JAVA_HOME=$(/usr/libexec/java_home -v 17)
```
## java 21 사용하기
```shell
export JAVA_HOME=$(/usr/libexec/java_home -v 21)
```
# jEnv를 이용한 Java 버전 관리
## jEnv 설치
```shell
brew install jenv
```
## shell 파일 수정
```shell
export PATH="$HOME/.jenv/bin:$PATH"
eval "$(jenv init -)"
```
## 설치된 jdk를 jenv에 추가한다.
```shell
jenv add /Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home
jenv add /Library/Java/JavaVirtualMachines/temurin-21.jdk/Contents/Home
```

## 자바 버전 변경
```shell
jenv global 17.0
```
# mac에서 설치된 자바 버전 확인하기
## 입력
```shell
/usr/libexec/java_home -V
```
## 아웃풋
```text
Matching Java Virtual Machines (5):
	24 (arm64) "Eclipse Adoptium" - "OpenJDK 24" ${java_version_path}
	21.0.6 (arm64) "Eclipse Adoptium" - "OpenJDK 21.0.6" ${java_version_path}
	17.0.12 (arm64) "Eclipse Adoptium" - "OpenJDK 17.0.12" ${java_version_path}
	17.0.7 (arm64) "Oracle Corporation" - "Java SE 17.0.7" ${java_version_path}
	15.0.10 (arm64) "Azul Systems, Inc." - "Zulu 15.46.17" ${java_version_path}
/Library/Java/JavaVirtualMachines/temurin-24.jdk/Contents/Home
```
