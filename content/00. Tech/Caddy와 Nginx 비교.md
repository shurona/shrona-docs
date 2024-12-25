---
tags:
  - nginx
  - caddy
---
# Nginx
## 특징
- 초기 성능
	- 고도로 최적화된 C 코드로 작성되어 매우 빠르다.
- 고부하 환경
	- 높은 트래픽 환경에서 더 우수한 성능을 보인다.
- 확장성
	- 고급 설정 및 모듈을 통해 더 높은 트래픽을 처리 가능하다.

>EC2에서  [[Nginx]]  적용하기
# Caddy
Caddy Server는 Go 언어로 작성된 오픈 소스 웹 서버입니다. Caddy는 간단하고 사용하기 쉬운 구성을 제공한다.
## 특징
- HTTPS를 기본으로 지원
	- Let's Encrypt와 같은 인증서 발급 및 갱신을 자동으로 처리한다.
- 초기 성능
	- 더 가벼운 구성으로 적은 트래픽에서는 빠르고 효율적이다.
-  멀티 코어 지원
	- Go 언어 기반으로 멀티 코어 활용이 뛰어나다.
- 확장성
	- 고성능을 유지하지만, 아주 높은 트래픽에서는 Nginx에 비해 한계가 있을 수 있다.

## EC2에서 Caddy 설치
```yaml
services:
  caddy:
    image: caddy:<version>
    restart: unless-stopped
    ports:
      - 80:80 # http 포트 열기
      - 443:443 # https 포트 열기
    volumes:
      - ${PWD}/Caddyfile:/etc/caddy/Caddyfile # 캐디 설정파일 복사
      - caddy_data:/data
      - caddy_config:/config
      - caddy_log:/var/log/caddy
    extra_hosts: # amazon linux에서 host.docker.internal 사용하기 위한 설정
      - "host.docker.internal:host-gateway"
    
  volumes:
    caddy_data:
    caddy_config:
    caddy_log:
    
```
### `caddy_data:/data`
Caddy 서버에서 TLS인증서나 기타 영구 데이터(캐시)와 같은 데이터를 저장하는데 사용한다.
### `caddy_config:/config`
Caddy의 설정 파일(Caddyfile 또는 JSON 설정)을 저장한다.
### `caddy_log:/var/log/caddy`
Caddy의 로그 파일을 저장한다.

## 기본 Caddyfile 작성
[기본 캐디 파일 공식 문서](https://caddyserver.com/docs/caddyfile/concepts)
```
domain.com {
        reverse_proxy http://host.docker.internal:{port} {
			header_up X-Real-Ip {remote_host}
        }
        log {
			output file /var/log/caddy/caddy_access_info.log {
				roll_size 100
				roll_local_time
				roll_keep 20
			}
        }
        encode gzip
}
```

- reverse_proxy `http://host.docker.internal:{port}`
	- 리버스 프록시를 설정한다.
- header_up X-Real-Ip {remote_host}
	- 프록시 요청에 실제로 어떤 IP가 들어오는 지 헤더를 추가한다. 
	- 외부에서 들어오는 클라이언트 IP를 헤더에 추가해주는 역할을 한다.
- log 설정
	- roll_size
		- 최대 크기를 100MB로 설정
	- roll_local_time
		- 타임스탬프를 로컬로 기록
	- roll_keep
		- 최대의 20개 파일을 유지한다.