## 개념
로그모니터링 시스템이면서 검색엔진이다. 오픈소스라서 무료로 사용이 가능하다

ELK 란 ELK Stack 이라고도 불리며 Elasticsearch, Logstash, Kibana 세 가지 오픈 소스 도구로 구성된 로그 및 데이터 분석 스택을 의미합니다.

데이터의 전송 흐름은 아래와 같다.
```mermaid
graph LR
    A["LogStash"] --> B["Elastic Search"]
    B --> C["Kibana"]
```

### ElasticSearch
- 분산 검색 및 실시간 데이터 분석 서비스이며 NoSQL입니다. 데이터는 문서(Document) 단위로 저장되며, 각 문서는 JSON 형식으로 저장되고 자동으로 인덱싱되어 효율적인 검색을 위해 최적화됩니다.

### Logstash

- 데이터 처리 파이프라인 도구로, 다양한 소스로부터 데이터를 수집, 변환, 필터링한 후 Elasticsearch나 다른 데이터 저장소로 전송합니다. ETL(Extract, Transform, Load) 도구 역할을 하며 실시간으로 로그 데이터를 처리하는 데 적합합니다.

### Kibana
- Elasticsearch 데이터를 시각화하는 대시보드 도구입니다. 사용자가 데이터를 시각적으로 탐색하고 분석할 수 있도록 다양한 그래프와 차트를 제공하며, 로그 모니터링, 메트릭 추적, 보안 분석 등의 대시보드를 쉽게 구축할 수 있습니다.
- 카프카 UI를 사용하는 것과 유사하다고 할 수 있다.


## Elastic Search를 사용하는 이유
MySQL와 같은 RDS에 비해서 부분 검색 성능이 NoSQL이 뛰어나서 사용을 한다.

### 조회 성능이 좋은 이유
Elastic Search는 데이터를 저장할 때 역 인덱스(또는 역색인) 방식을 사용합니다. ⇒ 키워드를 기반으로 문서를 정렬한다.

이 방식은 id가 아닌 키워드를 기준으로 데이터를 정렬하는 방식입니다. 그래서 키워드로 검색할 때, RDBMS의 Full Text Search처럼 문서 전체를 탐색하지 않아도 되어 더 빠르게 검색할 수 있습니다. 하지만 키워드가 많아지면 메모리 사용량이 크게 증가할 수 있다는 단점도 있습니다.

## LogStach의 Docker compose
#### LogStach를 위한 compose 설정파일
```yaml
logstash:
    build:
      context: logstash/
      args:
        ELASTIC_VERSION: ${ELASTIC_VERSION}
    volumes:
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro,Z
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro,Z
    ports:
      - 5044:5044
      - 50000:50000/tcp
      - 50000:50000/udp
      - 9600:9600
    environment:
      LS_JAVA_OPTS: -Xms256m -Xmx256m
      LOGSTASH_INTERNAL_PASSWORD: ${LOGSTASH_INTERNAL_PASSWORD:-}
    networks:
      - elk
    depends_on:
      - elasticsearch # 로컬에서 테스트 할 경우 필요한 elasticsearch
    restart: unless-stopped
```

**ELASTIC_VERSION**을 지정함으로써 **Elasticsearch와 LogStash의 버전**을 동기화하기 위함이다.
- ro
	- 읽기 전용으로 매핑합니다.
- Z
    - SELinux(리눅스 보안 모듈) 호환을 위해 사용합니다.

#### logstash.yml
```yaml
# 모니터링 및 API 인터페이스가 바인딩할 IP 주소를 지정한다.
# 아래의 주소는 모든 네트워크의 인터페이스에서 접근할 수 있도록 하였다.
http.host: 0.0.0.0

# LogStash 인스턴스의 노드 이름을 설정한다.
node.name: logstash
```
#### pipeline (logstash.conf)
```yaml
input {
# 포트 5044에서 Beats 데이터를 수신한다.
	beats {
		port => 5044
	}

# TCP 프로토콜을 통해서 데이터를 수신한다.
	tcp {
		port => 50000
		codec => json_lines
	}
}

output {
	# 콘솔 출력 형식을 설정
	stdout {
    	codec => rubydebug  
  }
  
  # elasticsearch를 위한 설정(로컬)
	elasticsearch {
		hosts => "elasticsearch:9200"
		index => "springboot-elk"
		user => "elastic"
		password => "pwd"
	}
}


```

## Elastic Cloud 설정
LogStash에서 elastic Cloud로 연결하기 위해서 Elastic Cloud [Elastic Cloud 사이트](ElasticElastic — The Search AI Company](https://www.googleadservices.com/pagead/aclk?sa=L&ai=DChcSEwjOgYuBqpqJAxXP1RYFHd09BV0YABAAGgJ0bA&co=1&ase=2&gclid=Cj0KCQjwsc24BhDPARIsAFXqAB2v_ru9vkwL9n9k2ogA367Oxanminp1nPh4hU-5EhowbKj6PIZyGy0aAjPNEALw_wcB&ohost=www.google.com&cid=CAESV-D2bHfylT_b10I2jpKaSJ4lc1ajQiYSfV_ey0s8yxq75EthXu3QJmkG4MD1Udl9WGn5GJIwDM6bFuT3JexlDUINTg_p2xlQowv-Pt1w7fVm2tYvR5d5qg&sig=AOD64_065gGRjMuGpmfWdWhNapiLRFyORg&q&nis=4&adurl&ved=2ahUKEwjNmYaBqpqJAxUtfPUHHa0QBtgQ0Qx6BAgaEAE) 로 들어가서 회원 가입 후 배포된 Depoyments를 확인해서 cloud_id 및 password를 갖고 온 후 아래에 기입해준다.
```yaml
output {
	elasticsearch {
		cloud_id => "${cloud_id}"
		ssl => true
		user => "elastic"
		password => "${password}"
		index => "%{index_name}"
	}
}
```

