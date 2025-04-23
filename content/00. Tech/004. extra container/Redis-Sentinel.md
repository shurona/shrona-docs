---
tags:
  - redis
  - redis/sentinel
---

# Redis Sentinel이란

[공식문서](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/)
클러스타가 아닌 환경에서 고가용성 제공을 위해서 모니터링, 알람, 자동 페일 오버를 제공하는 Redis용 고가용성 솔루션이다.
## 제공하는 기술
- 모니터링
	- 마스터 및 레플리카 인스턴스가 예상되로 작동하는지 지속적으로 확인한다.
- 알림
	- 모니터링 중인 Redis 인스턴스 중 하나에 문제가 발생하면 API를 통해서 시스템 관리자나 다른 컴퓨터에게 알림을 전달할 수 있다.
- 자동 페일오버
	- 만약 마스터에서 오류가 발생하면 다른 레플리카가 새로 마스터가 되도록 만들어주며, 레디스를 사용하는 애플리케이션이 연결할 때 새 주소에 대해서 정보를 받을 수 있도록 장애 조치 프로세스를 진행한다.
- 설정 전달
	- 클라이언트는 특정 서비스를 담당하고 있는 현재 Redis 마스터 주소를 요청하기 위해서 센티널에 연결한다.

## Sentinel을 사용한 분산 시스템의 장점
- 실패 탐지는 여러 Sentinel 마스터가 더 이상 사용할 수 없다는 것에 동의했을 때 발생한다.
- Sentinel 모든 Sentinel 프로세스가 작동하지 않더라도 작동하므로 장애에 대해 시스템을 견고하게 한다.

# 센티널을 배포하기 전에 기본적인 사항
- 단단한 서비스를 위해서는 최소 3개의 인스턴스를 배포해야 한다.
- 3개의 Sentinel 인스턴스는 독립적인 방식으로 장애가 발생할 것으로 예상되는 인스턴스에 배치되어야 한다. 
  만약 하나의 인스턴스가 다운되어서 모든 Sentinel이 다운되면 안되기 때문이다.
- Redis는 비동기 복제를 사용하기 때문에 Sentinel + Redis 분산 시스템은 장애가 발생하는 동안 acknowledged write(승인된 쓰기)를 보장하지 않는다.   
  그러나 손실되는 기간을 특정 순간으로 제한하는 Sentinel을 배포하는 방법이 있고 덜 안전한 배포 방법도 있다.
	- 클라이언트에서 사용되는 라이브러리가 센티널을  지원하는지 확인해야 한다.

## quorum(쿼럼)
- 센티널이 마스터가 장애로 간주하기 위한 Sentinel의 최소 숫자를 의미한다.
- 다만 실제로 장애 조치를 수행하려면 Sentinel중 한 개가 리더로 선출되서 장애 조치를 진행할 수 있는 권한을 얻어야 한다.

# 센티널 설정파일
```Text
// 일반적인 예시
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 60000
sentinel failover-timeout mymaster 180000
sentinel parallel-syncs mymaster 1
```

`sentinel monitor <master-name> <ip> <port> <quorum>`   
센티널이 모니터링을 할 마스터를 지정한다.   

`sentinel <option_name> <master_name> <option_value>`
- down-after-milliseconds
	- 마스터 노드가 장애로 간주되는 시간을 의미한다. (ms 단위)
- parallel-sync
	- 장애 조치 이후에 새 마스터를 사용하도록 재구성될 수 있는 복제본의 최대 개수를 동시에 설정합니다
		- 장애 발생 이후 새로 마스터가 선정이 되고 해당 마스터의 데이터를 복제본으로 복사를 할 때 진행하는 복제본의 개수를 의미한다.
	- 값이 낮을 수록 장애 조치 프로세스를 진행하는데 오래 걸린다.
		- 만약 10개의 복제본이 있다고 했을 때 1이면 한 개씩 동기화를 하므로 시간이 오래 걸리지만 대역폭을 아낄 수 있고 5개를 선정하면 5개씩 동기화를 하므로 시간은 금방 걸리지만 네트워크 및 서버의 과부하가 일어날 수 있다.

## 데이터 동기화를 위한 설정
min-replicas-to-write와 min-replicas-max-lag 설정을 이용해서 데이터 손실을 방지하고 비상정상적일 때 마스터의 쓰기를 중단할 수 있다. 
### 상세 정보
min-replicas-to-write
- 마스터가 쓰기 작업을 처리하기 위한 최소한의 복제본의 수

min-replicas-max-lag
 - 복제본이 마스터를 따라갈 때의 딜레이 시간(초)
 - 만약 해당 시간이 지나면 마스터는 복제본이 '정상'으로 간주하지 않는다.

# Redis Sentinel 적용
## Docker Compose
```
services:
  redis-master:
    image: redis:7.4.0
    container_name: redis-master
    command: [ "redis-server", "--requirepass", "masterpass" ]
    ports:
      - 6379:6379
    volumes:
      - redis-master-data:/data
    networks:
      - redis-network
    restart: always

  # Redis Slave 1
  redis-slave-1:
    image: redis:7.4.0
    container_name: redis-slave-1
    command: [ "redis-server", "--slaveof", "redis-master", "6379", "--masterauth", "masterpass", "--requirepass", "masterpass" ]
    ports:
      - 6380:6379
    volumes:
      - redis-slave-1-data:/data
    networks:
      - redis-network
    restart: always

  # Redis Slave 2
  redis-slave-2:
    image: redis:7.4.0
    container_name: redis-slave-2
    command: [ "redis-server", "--slaveof", "redis-master", "6379", "--masterauth", "masterpass", "--requirepass", "masterpass" ]
    ports:
      - 6381:6379
    volumes:
      - redis-slave-2-data:/data
    networks:
      - redis-network
    restart: always

  # Sentinel 1
  redis-sentinel-1:
    image: redis:7.4.0
    container_name: redis-sentinel-1
    command: [ "redis-sentinel", "/etc/sentinel.conf" ]
    depends_on:
      - redis-master
      - redis-slave-1
      - redis-slave-2
    volumes:
      - ./sentinel.conf:/etc/sentinel.conf
    networks:
      - redis-network
    ports:
      - 26379:26379
    restart: always

  # Sentinel 2
  redis-sentinel-2:
    image: redis:7.4.0
    container_name: redis-sentinel-2
    command: [ "redis-sentinel", "/etc/sentinel.conf" ]
    depends_on:
      - redis-master
      - redis-slave-1
      - redis-slave-2
    volumes:
      - ./sentinel.conf:/etc/sentinel.conf
    networks:
      - redis-network
    ports:
      - 26380:26379
    restart: always

  # Sentinel 3
  redis-sentinel-3:
    image: redis:7.4.0
    container_name: redis-sentinel-3
    command: [ "redis-sentinel", "/etc/sentinel.conf" ]
    depends_on:
      - redis-master
      - redis-slave-1
      - redis-slave-2
    volumes:
      - ./sentinel.conf:/etc/sentinel.conf
    networks:
      - redis-network
    ports:
      - 26381:26379
    restart: always

volumes:
  redis-master-data:
  redis-slave-1-data:
  redis-slave-2-data:

networks:
  redis-network:
```

### 주요 설명
- Redis Master 및 다른 Slaves 들의 비밀번호를 설정해준다. 
	- `command: [ "redis-server", "--requirepass", "masterpass" ]`
- 지정된 Redis를 Master의  slave로 설정해 준다.
	- `[ "redis-server", "--slaveof", "redis-master", "6379", "--masterauth", "masterpass", "--requirepass", "slavepass" ]`
- Redis Sentinel은 conf를 설정해서 sentinel이 시작될 때 전달되게 해준다.


## sentinel conf 파일
```conf
# 내부에서는 하나의 port를 사용하는 것을 의미한다.
port 26379

# Sentinel이 redis-master와 같은 호스트 이름을 해석할 수 있도록 설정
sentinel resolve-hostnames yes

# Sentinel이 mymaster라는 이름의 Redis 마스터를 모니터링 설정
# 2는 장애 발생을 합의하기 위한 숫자를 의미한다.
sentinel monitor mymaster redis-master 6379 2

# Redis Master와 통신할 때 사용할 비밀번호
sentinel auth-pass mymaster masterpass

# 마스터가 장애 상태임을 확인하기 위한 시간
sentinel down-after-milliseconds mymaster 5000

# 장애 조치를 완료하는 데 필요한 최대 시간
sentinel failover-timeout mymaster 10000

# 장애 이후에 마스터가 슬레이브와 동기화를 진행할 때 진행하는 숫자
sentinel parallel-syncs mymaster 1
```

## Master와 Slave의 연결상태를 확인
```text
# Master 정보
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=172.20.0.2,port=6379,state=online,offset=53330,lag=0
slave1:ip=172.20.0.4,port=6379,state=online,offset=53330,lag=0
master_failover_state:no-failover
master_replid:3a0372a9c0e737779adcb625de18dcd803c6daa3
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:53465
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:53465


# Slave 정보
# Replication
role:slave
master_host:redis-master
master_port:6379
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_read_repl_offset:567791
slave_repl_offset:567791
slave_priority:100
slave_read_only:1
replica_announced:1
connected_slaves:0
master_failover_state:no-failover
.
.
.
```
위의 replication의 command를 이용해서 마스터에서 slave의 정보를 확인할 수 있다.    
반대로 Slave에서도 Master을 확인할 수 있음을 볼 수 있다.   
위에서 보면 ip와 port 정보를 얻을 수 있는 것을 보여준다.

## sentinel에서 연결정보 확인하기
```text
127.0.0.1:26379> sentinel master mymaster
 1) "name"
 2) "mymaster"
 3) "ip"
 4) "172.20.0.2"
 5) "port"
 6) "6379"
 7) "runid"
 8) "9b762135a3bf2f062f47043a90a0c9c218cc4b86"
 9) "flags"
10) "master"
11) "link-pending-commands"
12) "0"
13) "link-refcount"
14) "1"
15) "last-ping-sent"
16) "0"
17) "last-ok-ping-reply"
18) "131"
19) "last-ping-reply"
20) "131"
21) "down-after-milliseconds"
22) "5000"
23) "info-refresh"
24) "9392"
25) "role-reported"
26) "master"
27) "role-reported-time"
28) "99817"
29) "config-epoch"
30) "0"
31) "num-slaves"
32) "2"
33) "num-other-sentinels"
34) "2"
35) "quorum"
36) "2"
37) "failover-timeout"
38) "10000"
39) "parallel-syncs"
40) "1"


127.0.0.1:26379> SENTINEL get-master-addr-by-name mymaster
1) "172.20.0.2"
2) "6379"


// slave의 상태도 아래의 cli를 사용해서 확인할 수 있다.
127.0.0.1:26379> sentinel slaves mymaster

```
위와 같이 master에 대한 ip 주소 뿐만 아니라 slave와 같은 정보 들도 얻어올 수 있다.

# 트러블 슈팅
## Sentinel에서 slave로 접근하지 못하는 문제
### 문제 상황
redis master와 slave를 다른 비밀번호로 설정한 다음에 sentinel config를 위와 같이 구성을 했더니   
sentinel에서 아래와 같이 slave를 연결하지 못하는 문제가 발생하였다.
```text
1) "master-link-down-time"
2) "0"
3) "master-link-status"
4) "err"
5) "master-host"
6) "?"
7) "master-port"
8) "0"
```

### 해결방법
1. master와 slave redis의 비밀번호를 통일하면 위의 문제는 해결이 되는 것을 확인할 수 있다.

## Master를 죽인 다음에 hostname을 찾는 로그가 남는 이유
### 문제 상황
테스트 하는 도중에 Redis Master를 죽인 이후에 Slave에서 Master로 성공적으로 변환이 되어도    
`Failed to resolve hostname`가 남아있었는데 이렇게 계속해서 로그 발생하는 원인을 찾아보았다.
### 해결 방법
그 이유는 sentinel이 마스터가 죽더라도  그 정보는 계속해서 갖고 있어서 죽은 이후에도 다시 살아날 경우 연결하기 위해서 찾고 있었기 때문에 발생하는 로그였다.   
만약 로그를 멈추고 싶으면
`sentinel reset [masterName]`을 입력해서 초기화 해주면 된다. 
#### 공식문서 발췌
```text
### Removing the old master or unreachable replicas

Sentinels never forget about replicas of a given master, even when they are unreachable for a long time. This is useful, because Sentinels should be able to correctly reconfigure a returning replica after a network partition or a failure event.

Moreover, after a failover, the failed over master is virtually added as a replica of the new master, this way it will be reconfigured to replicate with the new master as soon as it will be available again.

However sometimes you want to remove a replica (that may be the old master) forever from the list of replicas monitored by Sentinels.

In order to do this, you need to send a `SENTINEL RESET mastername` command to all the Sentinels: they'll refresh the list of replicas within the next 10 seconds, only adding the ones listed as correctly replicating from the current master [`INFO`](https://redis.io/commands/info) output.
```

## Master를 죽인 이후 다시 살렸을 때 재연결이 안되는 오류
### 문제 상황
Master의 작동을 멈춘다음에 slave 중 하나가 Master가 된 이후에 다시 이전에 Master를 실행했더니 재연결이 안되는 오류가 발생하였다.   
`NOAUTH Authentication required` 에러 발생
### 해결 방법
기존의 redis-master 코드를 아래와 같이 구성을 하고 있었는데 재 연결 이후 해당 마스터는 slaves가 되는데 마스터의 비밀번호를 몰라서 발생하는 에러였다.   
아래와 같이 redis server를 실행할 때 master auth도 같이 넣어줌으로써 문제를 해결할 수 있었다.
```yaml
redis-master:
    image: redis:7.4.0
    container_name: redis-master
    command: ["redis-server","--requirepass","masterpass"]
    .
    .
    .

# command를 아래와 같이 수정
=> command: ["redis-server","--masterauth","masterpass","--requirepass","masterpass"]
```