---
tags:
  - cloudwatch
  - ec2
  - monitoring
---
# EC2 기본 모니터링
- EC2 인스턴스는 기본적으로 CPU, 네트워크, 디스크 I/O 등 일부 지표만 CloudWatch로 전송한다
- 메모리 사용량 등은 기본 지표에 포함되어 있지 않아, 별도의 설정이 필요하다.
# IAM 역할 연결
CloudWatch Agent가 메트릭을 전송하려면 EC2 인스턴스에 적절한 IAM 역할이 연결되어 있어야 한다.   
역할이 연결이 안되어 있을 때 아래와 같은 오류가 발생했다.
```text
"error":"NoCredentialProviders: no valid providers in chain
caused by: EnvAccessKeyNotFound: failed to find credentials in the environment.
```
### 방법
- IAM 역할 생성
	- AWS 콘솔에서 IAM > 역할 > 역할 만들기
	- `CloudWatchAgentServerPolicy`정책 연결해서 역할 생성
- EC2 인스턴스에 직접 연결
	- EC2 콘솔에서 인스턴스 선택 → 보안 → IAM 역할 수정
	- 위에서 만든 IAM 역할을 선택해 연결
# CloudWatch Agent 설치 및 설정
- OS에 맞는 에이전트 설치
	- ec2 amazon linux는 아래와 같이 설치가 가능하다.
```shell
sudo yum install amazon-cloudwatch-agent

# 이후 마법사를 열어서 설정 파일을 생성해 준다.
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```
- 원하는 설정에 맞게 선택해주면 아래와 같은 config 파일이 생성된다.
- `/opt/aws/amazon-cloudwatch-agent/bin/config.json` 생성 위치
```json
{
	"agent": {
		"metrics_collection_interval": 60,
		"run_as_user": "root"
	},
	"metrics": {
		"aggregation_dimensions": [
			[
				"InstanceId"
			]
		],
		"append_dimensions": {
			"AutoScalingGroupName": "${aws:AutoScalingGroupName}",
			"ImageId": "${aws:ImageId}",
			"InstanceId": "${aws:InstanceId}",
			"InstanceType": "${aws:InstanceType}"
		},
		"metrics_collected": {
			"cpu": {
				"measurement": [
					"cpu_usage_idle",
					"cpu_usage_iowait",
					"cpu_usage_user",
					"cpu_usage_system"
				],
				"metrics_collection_interval": 60,
				"totalcpu": false
			},
			"disk": {
				"measurement": [
					"used_percent",
					"inodes_free"
				],
				"metrics_collection_interval": 60,
				"resources": [
					"*"
				]
			},
			"diskio": {
				"measurement": [
					"io_time"
				],
				"metrics_collection_interval": 60,
				"resources": [
					"*"
				]
			},
			"mem": {
				"measurement": [
					"mem_used_percent"
				],
				"metrics_collection_interval": 60
			},
			"swap": {
				"measurement": [
					"swap_used_percent"
				],
				"metrics_collection_interval": 60
			}
		}
	}
}

```

- 생성된 json을 적절한 위치로 옮겨서 agent를 실행한다.
	- 이번에 설정할 때는 `/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json`로 위치해야 했다.
```shell
sudo systemctl restart amazon-cloudwatch-agent
```
- 실행한 이후에 실행 상태를 확인해준다.
```shell
sudo systemctl status amazon-cloudwatch-agent

# 로그는 여기서 확인이 가능하다.
cat /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```
- runnging으로 현재 실행 중인 것을 확인할 수 있다.
```text
● amazon-cloudwatch-agent.service - Amazon CloudWatch Agent
     Loaded: loaded (/etc/systemd/system/amazon-cloudwatch-agent.service; disabled; preset: disabled)
     Active: active (running) since Tue 2025-06-10 23:36:42 UTC; 0h 18min ago
   Main PID: 410 (amazon-cloudwat)
      Tasks: 7 (limit: 1114)
     Memory: 57.0M
        CPU: 26.936s
```

# CloudWatch 콘솔에서 확인
- Cloudwatch -> 모든 지표 -> CWAgent(기본 값) -> 설정한 ec2의 인스턴스 아이디를 확인 하면 메트릭 정보를 확인할 수 있다.
- 예시
	- mem_used_percent
	- swap_used_percent
- 이후 원하는 임계치에 따라서 알람 설정이 가능해진다.
# 진행 중 발생한 문제
- 인증 정보 관련 에러
	- `EnvAccessKeyNotFound: failed to find credentials in the environment`
		- IAM 역할이 인스턴스에 연결되어 있지 않거나, 권한이 부족한 경우 발생
		- 역할 연결 및 권한 정책(`CloudWatchAgentServerPolicy`) 확인
- 매트릭이 안보일 때
	- 콘솔에서 CWAgent 네임스페이스를 제대로 보고 있는지 확인
	- 모든 지표 이후 EC2가 아닌 모두 상태에서 확인해야 한다.