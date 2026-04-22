1. PC2: 카프카 서버 설치 및 설정 (10.15.0.170)
먼저 데이터를 받아줄 카프카 엔진을 PC2에 설치해야 합니다. Rocky Linux 기준입니다.

1단계: Java 설치
카프카는 Java 위에서 동작합니다.

Bash
sudo dnf install java-11-openjdk-devel -y
2단계: 카프카 다운로드 및 압축 해제
Bash
wget https://downloads.apache.org/kafka/3.6.0/kafka_2.13-3.6.0.tgz
tar -xzf kafka_2.13-3.6.0.tgz
cd kafka_2.13-3.6.0
3단계: 외부 접속을 위한 설정 변경
PC1에서 접속할 수 있도록 설정 파일(config/server.properties)을 수정해야 합니다.

Bash
vi config/server.properties
수정 사항: listeners와 advertised.listeners 부분을 찾아 아래와 같이 수정합니다. (주석 # 제거)

listeners=PLAINTEXT://0.0.0.0:9092

advertised.listeners=PLAINTEXT://10.15.0.170:9092

4단계: 실행 (Zookeeper & Kafka)
터미널을 두 개 열어 각각 실행합니다.

Bash
# 터미널 1: 주키퍼 실행
bin/zookeeper-server-start.sh config/zookeeper.properties

# 터미널 2: 카프카 서버 실행
bin/kafka-server-start.sh config/server.properties
2. PC1: 파일 전송용 Producer 코드 (10.7.0.2)
PC1은 지정된 디렉토리를 감시하다가 파일을 카프카로 쏘아 올리는 역할을 합니다.

라이브러리 설치
Bash
pip install kafka-python
Producer 파이썬 코드 (producer.py)
이 코드는 지정된 폴더 내의 파일을 읽어 바이트 형태로 카프카 토픽(file-transfer)에 전송합니다.

Python
import os
from kafka import KafkaProducer

# 설정
KAFKA_SERVER = '10.15.0.170:9092'
TOPIC_NAME = 'file-transfer'
WATCH_DIR = '/path/to/your/source/directory' # 보낼 파일이 있는 디렉토리

producer = KafkaProducer(
    bootstrap_servers=[KAFKA_SERVER],
    max_request_size=52428800  # 최대 전송 크기 설정 (예: 50MB)
)

def send_file(file_path):
    file_name = os.path.basename(file_path)
    with open(file_path, 'rb') as f:
        file_data = f.read()
        # 헤더에 파일명을 넣어 전송
        producer.send(TOPIC_NAME, value=file_data, key=file_name.encode('utf-8'))
    producer.flush()
    print(f"성공적으로 전송됨: {file_name}")

if __name__ == "__main__":
    for root, dirs, files in os.walk(WATCH_DIR):
        for file in files:
            # 새로 생긴 파일 로직 (실제 서비스에선 DB나 로그로 전송 여부 체크 권장)
            full_path = os.path.join(root, file)
            send_file(full_path)
3. PC2: 파일 수신용 Consumer 코드 (10.15.0.170)
PC2에서 카프카 메시지를 기다리고 있다가, 메시지가 오면 파일로 다시 저장합니다.

Consumer 파이썬 코드 (consumer.py)
Python
import os
from kafka import KafkaConsumer

# 설정
KAFKA_SERVER = 'localhost:9092'
TOPIC_NAME = 'file-transfer'
SAVE_DIR = '/path/to/save/directory' # 저장할 디렉토리

if not os.path.exists(SAVE_DIR):
    os.makedirs(SAVE_DIR)

consumer = KafkaConsumer(
    TOPIC_NAME,
    bootstrap_servers=[KAFKA_SERVER],
    auto_offset_reset='earliest',
    group_id='file-receiver-group',
    fetch_max_bytes=52428800,
    max_partition_fetch_bytes=52428800
)

print("파일 수신 대기 중...")

for message in consumer:
    file_name = message.key.decode('utf-8')
    file_data = message.value
    
    save_path = os.path.join(SAVE_DIR, file_name)
    with open(save_path, 'wb') as f:
        f.write(file_data)
    
    print(f"파일 저장 완료: {save_path}")
4. 스케줄링 설정 (Cron)
매일 밤 12시와 오전 11시에 실행되도록 PC1의 crontab에 등록합니다.

Bash
crontab -e
아래 내용을 추가합니다:

Bash
0 0 * * * /usr/bin/python3 /path/to/producer.py
0 11 * * * /usr/bin/python3 /path/to/producer.py
💡 유의사항 및 조언
파일 크기 제한: 카프카는 기본적으로 메시지 당 1MB 제한이 있습니다. 위 코드에서 max_request_size를 늘렸지만, 카프카 서버(server.properties)의 message.max.bytes 설정도 함께 늘려줘야 대용량 파일 전송이 가능합니다.

중복 전송 방지: 위 예제 코드는 실행될 때마다 폴더 내 모든 파일을 보냅니다. 이미 보낸 파일인지 확인하려면 파일의 수정 시간(mtime)을 체크하거나, 전송 완료된 파일을 archive 폴더로 이동시키는 로직을 추가하는 것이 좋습니다.

네트워크 방화벽: PC2의 9092 포트가 PC1으로부터의 접속을 허용하도록 방화벽(firewalld) 설정을 확인해 주세요.
