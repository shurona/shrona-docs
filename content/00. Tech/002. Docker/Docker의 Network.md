---
tags:
  - docker/network
---
[공식문서](https://docs.docker.com/engine/network/#network-driver-summary)
# 개요
컨테이너 네트워킹은 컨테이너가 서로 연결하고 통신하는 능력을 의미한다.   
컨테이너들은 기본적으로 네트워킹이 가능하며 나가는 연결을 만들 수 있다.   
컨테이너에는 기본적으로 어떤 종류의 네트워크에 붙어있는지 동료(peer)가 도커 워크로드인지 아닌지에 대한 정보가 없다.   
컨테이너는 IP 주소, 게이트웨이, 라우팅 테이블, DNS 서비스 및 기타 네트워킹의 세부 사항만을 바라본다.
# User-defined networks
사용자가 임의로 정의한 네트워크를 생성하고 여러 컨테이너를 동일하나 네트워크에 연결 할 수 있다. 
```shell
docker network create -d bridge my-net
docker run --network=my-net -itd --name=container3 busybox
```
## Drives
아래의 네트워크 드라이버는 기본적으로 사용할 수 있는 핵심(core) 기능들을 제공한다.

| Driver    | Description                                                                                                                  |
| --------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `bridge`  | 기본 네트워크 드라이버.                                                                                                                |
| `host`    | Remove network isolation between the container and the Docker host.                                                          |
| `none`    | Completely isolate a container from the host and other containers.                                                           |
| `overlay` | Overlay networks connect multiple Docker daemons together.<br>오버레이 네트워크는 여러 도커 데몬을 함께 연결합니다.                                 |
| `ipvlan`  | IPvlan networks provide full control over both IPv4 and IPv6 addressing.<br>IPVLAN 네트워크는 IPv4 및 IPv6 주소 지정을 완전히 제어 할 수 있습니다. |
| `macvlan` | Assign a MAC address to a container.<br>컨테이너에 Mac 주소를 할당하십시오.                                                                |

# Container Networks
사용자 정의 네트워크 외에도 -network 컨테이너를 사용하여서 다른 컨테이너 네트워킹 스택에 직접 연결할 수도 있다.

아래의 예제는 Redis 컨테이너를 실행하고 Redis가 localhost에 바인딩 한 다음 redis-cli 명령어를 실행을 하고 localhost 인터페이스를 통해서 redis 서버에 연결을 한다.
```shell
docker run -d --name redis example/redis --bind 127.0.0.1
docker run --rm -it --network container:redis example/redis-cli -h 127.0.0.1
```

# Published ports
기본적으로 Docker Create 또는 Docker Run을 사용하여 컨테이너를 생성하거나 실행할 때 브리지 네트워크의 컨테이너는 포트를 외부에 노출시키지 않습니다.   
-publish 또는 -p 플래그를 사용하여 브리지 네트워크를 외부의 서비스에 포트와 매핑해서 사용할 수 있다.   
이것은 호스트에서 방화벽 규칙을 생성하여 컨테이너 포트를 Docker 호스트의 포트에 외부로 매핑합니다.

|Flag value|Description|
|---|---|
|`-p 8080:80`|Map port `8080` on the Docker host to TCP port `80` in the container.|
|`-p 192.168.1.100:8080:80`|Map port `8080` on the Docker host IP `192.168.1.100` to TCP port `80` in the container.|
|`-p 8080:80/udp`|Map port `8080` on the Docker host to UDP port `80` in the container.|
|`-p 8080:80/tcp -p 8080:80/udp`|Map TCP port `8080` on the Docker host to TCP port `80` in the container, and map UDP port `8080` on the Docker host to UDP port `80` in the container.|