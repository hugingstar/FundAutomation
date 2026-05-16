# Data into provisioning node from Airflow worker node

(가정) 10.0.0.0/8 네트워크 상에서 Kafka를 사용하여 파일을 전송한다.
- Producer : 10.7.0.2/8
- Consumer : 10.15.0.170/8

## Kafka server
- Consumer side에 서버를 설치한다.(별도로 구성해도 된다.)
### 1.카프카 서버 설치 및 설정(Consumer side)
- 먼저 데이터를 받아줄 카프카 엔진을 PC2에 설치 
- Rocky Linux/Ubuntu 기준 설치 방법

```
sudo dnf install java-11-openjdk-devel -y

# Ubuntu
sudo apt update
sudo apt install openjdk-11-jdk -y
java -version
```

### 2.카프카 다운로드 및 압축 해제(Consumer side)

```
# 3.7.0 버전
wget https://archive.apache.org/dist/kafka/3.7.0/kafka_2.13-3.7.0.tgz
tar -xzf kafka_2.13-3.7.0.tgz
cd kafka_2.13-3.7.0
```

### 3. 외부 접속을 위한 설정 변경(Consummer side)
- PC1에서 접속할 수 있도록 설정 파일(config/server.properties)을 수정해야 합니다.
- Rocky/Ubuntu 동일
```
vi config/server.properties
```

- 수정 사항: listeners와 advertised.listeners 부분을 찾아 아래와 같이 수정합니다. (주석 # 제거)
- 포트를 사용해야 하므로 IP 주소를 수정해준다.
```
listeners=PLAINTEXT://0.0.0.0:9092
# Consumer side 주소
advertised.listeners=PLAINTEXT://10.15.0.170:9092
```

### 4. 실행 (Zookeeper & Kafka)
- Consumer side에서 터미널을 두 개 열어 각각 실행합니다.
- 현재 작업 위치 : kafka_2.13-3.7.0

- **터미널 1**: 주키퍼 실행
```
bin/zookeeper-server-start.sh config/zookeeper.properties
```

- **터미널 2**: 카프카 서버 실행
```
bin/kafka-server-start.sh config/server.properties
```

## Producer side
### 1. 파일 전송용 Producer 코드
- Producer side는 지정된 디렉토리를 감시하다가 파일을 카프카로 쏘아 올리는 역할을 한다.

### 2. 라이브러리 설치
```
sudo apt install python3-pip -y
pip install kafka-python
pip install lz4
```

### 3. 설정
- Producer.py : 지정된 폴더 내의 파일을 읽어 바이트 형태로 카프카 토픽(file-transfer)에 전송합니다.

```
KAFKA_SERVER = '10.15.0.170:9092' # Kafka server IP (Consumer side에 설치)
TOPIC_NAME = 'file-transfer'
WATCH_DIR = '/path/to/your/source/directory' # 보낼 파일이 있는 디렉토리

producer = KafkaProducer(
    bootstrap_servers=[KAFKA_SERVER],
    max_request_size=52428800  # 최대 전송 크기 설정 (예: 50MB)
)
```

## Consumer side 
### 1. 파일 수신용 Consumer
- Consumer PC에서 카프카 메시지를 기다리고 있다가, 메시지가 오면 파일로 다시 저장합니다.

# 2. 설정
```
KAFKA_SERVER = 'localhost:9092' # 같은 곳에 설치했기 때문에 localhost(별도 구성시에는 바꿔줘야 한다.)
TOPIC_NAME = 'file-transfer'
SAVE_DIR = '/path/to/save/directory' # 저장할 디렉토리

consumer = KafkaConsumer(
    TOPIC_NAME,
    bootstrap_servers=[KAFKA_SERVER],
    auto_offset_reset='earliest',
    group_id='file-receiver-group',
    fetch_max_bytes=52428800,
    max_partition_fetch_bytes=52428800
)
```

## 4. Crontab
- 분석 결과 완료시간 or 서비스 론칭 시간 고려하여 설정
- 매일 밤 12시와 오전 11시에 실행되도록 Consumer의 crontab에 등록
- Producer에서 crontab을 설정해야 한다.
- Consumer는 대기중... 

```
# 수정할 파일
crontab -e

0 0 * * * /usr/bin/python3 /path/to/producer.py
0 11 * * * /usr/bin/python3 /path/to/producer.py
```

💡 유의사항 및 조언
- **파일 크기 제한**: 카프카는 기본적으로 메시지 당 1MB 제한이 있다. 위 코드에서 max_request_size를 늘렸지만, 카프카 서버(server.properties)의 message.max.bytes 설정도 함께 늘려줘야 대용량 파일 전송이 가능하다.

- **중복 전송 방지**: 위 예제 코드는 실행될 때마다 폴더 내 모든 파일을 보냅니다. 이미 보낸 파일인지 확인하려면 파일의 수정 시간(mtime)을 체크하거나, 전송 완료된 파일을 archive 폴더로 이동시키는 로직을 추가하는 것이 좋다.

- **네트워크 방화벽**: Consumer PC의 9092 포트가 Producer PC으로부터의 접속을 허용하도록 방화벽(firewalld) 설정을 확인


## 5. 전송된 파일 갯수가 일치하지 않을 때
- Producer와 Consumer에서 파일 갯수를 확인한다.

```
ls -l | grep ^- | wc -l
```

- 전송하는 파일이 끊기지 않도록 Kafka 서버에서 메시지 최대 용량을 설정해줘야 한다.

```
# {kafka 설치 폴더}/config/server.properties
message.max.bytes=104857600        # 100MB
replica.fetch.max.bytes=104857600  # 100MB
group.max.session.timeout.ms=300000 # 60s 보다 크게 설정
```

- Producer 부분에서도 수정해줘야 한다.
- MANIFEST_TIMEOUT, MAX_FILE_SIZE 부분을 적용한다.

```
BOOTSTRAP_SERVERS = ['10.22.0.205:9092'] # 카프카 서버 주소
TOPIC_DATA = 'market_data_sync'
TOPIC_MANIFEST_REQUEST = 'market_data_manifest_request'
TOPIC_MANIFEST_RESPONSE = 'market_data_manifest_response'
BASE_DIR = '/root/Data'
MARKETS = ['KOSPI']
SHEETS = ['A1Sheet', 'B1Sheet', 'C1Sheet']
MANIFEST_TIMEOUT = 120  # 1000개 파일 해시 계산 여유 시간
MAX_FILE_SIZE = 100 * 1024 * 1024  # 100MB
```

- Consumer 부분에서도 적용한다. 
- HASH_WORKERS, MAX_FETCH_BYTES를 상황에 따라서 수정한다.

```
BOOTSTRAP_SERVERS = ['10.22.0.205:9092'] # 카프카 서버 주소
TOPIC_DATA = 'market_data_sync'
TOPIC_MANIFEST_REQUEST = 'market_data_manifest_request'
TOPIC_MANIFEST_RESPONSE = 'market_data_manifest_response'
TARGET_BASE_DIR = '/root/Data'
HASH_WORKERS = 8
MAX_FETCH_BYTES = 100 * 1024 * 1024  # 100MB — Producer max_request_size와 통일
```
- auto_offset_reset='earliest' + rebalance 조합으로 돌리면 1000개를 보내야 하는데, 200개씩 끊기는 문제가 발생한다.
-  해결 방법 : poll 방식 + 비동기 저장 + 수동 offset 커밋
