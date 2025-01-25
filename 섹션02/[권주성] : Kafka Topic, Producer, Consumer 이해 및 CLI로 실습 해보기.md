# 섹션 2 Kafka Topic, Producer, Consumer 이해 및 CLI로 실습 해보기

## Kafka란 무엇인가?

![CleanShot 2025-01-25 at 21 25 16](https://github.com/user-attachments/assets/3fca79e7-6130-45e5-9ee5-b08345b8a9b9)

- **`오픈 소스 분산 이벤트 스트림 플랫폼`**
- 특징
  - `분산 환경`
    - 고가용성, 안정성, 확장성
    - 스케일 아웃
    - 복제
  - `이벤트 스트림`
    - 기존 메시지와는 다른 스케일의 데이터를 처리할 수 있음
    - 빅데이터
    - 실시간 
  - `오픈 소스 플랫폼`
    - 계속 유지 및 발전 중
    - 생태계가 존재함
      - Connector, KsqlDB, Schema Registry등등
    - Confluent, MSK, RedPanda, 등의 엔터프라이즈 매니지드 클라우드 서비스가 존재함
    - 많은 회사에서 채택해서 사용중

## Kafka 아키텍처

![image (3)](https://github.com/user-attachments/assets/fd66e8dc-a527-4409-b507-39aa5d39dd24)

- **주요 구성 요소**
  - `프로듀서(Producer)`
    - 데이터를 생성해 브로커에 전송하는 역할
    - 특정 토픽/파티션에 메시지를 전송할 수 있음
    - 키를 기반으로 파티션 분배 제어 가능
    - 동기/비동기 전송 지원
  - `컨슈머(Consumer)`
    - 브로커에서 데이터를 읽어 소비하는 역할
    - 컨슈머 그룹을 통해 병렬로 처리 가능
    - 오프셋 기반으로 처리 상태를 관리하여 재처리하거나 중복을 방지할 수 있음
  - `브로커(Broker)`
    - 카프카 클러스터 내 노드로서의 요소
    - 데이터를 저장하고 관리하며 프로듀서와 컨슈머 
	  - 프로듀서의 데이터를 수신해 디스크에 저장
	  - 컨슈머에게 요청 시 데이터를 제공
	  - 클러스터 내 데이터 분산으로 확장성과 내구성 보장
  - `토픽(Topic)`
    - 데이터 스트림을 구분하는 논리적 단위
    - 여러 컨슈머 그룹이 동시에 구독 가능
    - 파티션으로 세분화해 병렬 처리 지원
  - `파티션(Partition)`
    - 토픽의 데이터를 분산 저장하는 물리적 단위
    - 데이터 병렬 처리 및 확장 지원
    - 각 파티션은 리더(읽기/쓰기)와 팔로워(복제본)로 구성
    - 각 메시지(커밋 로그)는 오프셋으로 순서 관리
  - `주키퍼(Zookeeper)`
    - 카프카 클러스터를 안정적으로 운영할 수 있도록 하는 역할
    - 클러스터의 메타데이터 관리와 조율
      - 브로커, 파티션, 토픽 상태 추적
      - 리더-팔로워 관리 및 장애 복구 지원
    - 최신 카프카 2.x 이상 버전에서는 메타데이터 관리가 카프카 자체로 대체되어 분리되어 사라지는 추세

### Topic과 Partition, Offset

![img](https://github.com/user-attachments/assets/83e8959a-048c-4a92-a8ce-4d2136318ad6)

- **`Topic은 Partition으로 구성된 일련의 로그 파일의 집합체`**
  - RDBMS의 Table과 유사
  - Event, Message는 file이며 Topic은 folder
- 즉, 데이터를 논리적으로 구분하기 위한 개념적 단위

- Topic은 시간의 흐름에 따라 메시지가 순차적으로 물리적인 파일에 write됨

- `Topic의 Partition은 Kafka의 병렬 성능과 가용성 기능의 핵심 요소`
  - 메시지는 병렬 성능과 가용성을 고려하여 개별 파티션에 분산 저장됨
  - 여기서 Event(Message) 는 Key와 Value 기반의 메시지 구조이며, Value로 어떤 타입의 메시지도 가능함(문자열, 숫자값, 객체, Json, Avro, Protobuf등)

- Topic은 1개 이상의 파티션을 가질 수 있음
- Topic의 Partition들은 단일 카프카 브로커 뿐만 아니라 여러 개의 카프카 브로커 들에 분산 저장

## Producer & Consumer
- Producer는 Topic에 메시지를 보냄
  - Producer는 성능/로드밸런싱/가용성/업무 정합성등을 고려하여 어떤 브로커의 파티션으로 메시지를 보내야 할지 전략적으로 결정됨
  - 어떤 파티션으로 보낼지에 대한 책임을 프로듀서가 가짐

  ![image (5)](https://github.com/user-attachments/assets/d284d58d-1142-4686-a5fc-35bdce52418a)

- Consumer는 Topic에서 메시지를 읽어 들임
  - 여러 개의 Consumer들로 구성될 경우 어떤 브로커의 파티션에서 메시지를 읽어들일지 전략적으로 결정함
  - Pulling 방식으로 주기적으로 브로커의 토픽 파티션에서 메시지를 읽어들이며, 메시지를 성공적으로 가져 왔으면 commit을 통해서 __consumer_offse에 다음에 읽을 offset 위치를 기재함

## Console 기반 실습
- Docker-Compose 로 Confluent 7.7.0 설치
- 카프카에서 기본적으로 제공하는 디버깅 용도의 kafka-console-producer와 kafka-console-consumer로 실습 진행
- Console상에서 comman를 통해 producer의 메시지 전송과 consumer의 메시지 읽기 수행

### Topic 
```
// topic 생성
kafka-topics --bootstrap-server localhost:9092 --create --topic test_topic --partitions 3 --replication-factor 2

// topic 리스트 조회
kafka-topics --bootstrap-server localhost:9092 --list

// topic 토픽 상세 정보 조회
kafka-topics --bootstrap-server localhost:9092 --topic test_topic --describe

// topic 토픽 삭제
kafka-topics --bootstrap-server localhost:9092 --topic test_topic --delete
```

### Producer & Consumer
```
// kafka-console-producer로 메시지 보내기
kafka-console-producer --bootstrap-server localhost:9092 --topic test-topic

// kafka-console-consumer로 메시지 읽기
kafka-console-consumer --bootstrap-server localhost:9092 --topic test-topic --from-beginning

// 동일 키는 동일 파티션

// key 가 있는 message 보내기
kafka-console-producer --bootstrap-server localhost:9092 --topic test-topic \
--property key.separator=: --property parse.key=true

// key 가 있는 message 읽기
kafka-console-consumer --bootstrap-server localhost:9092 --topic test-topic \
--property print.key=true --property print.value=true --property print.partition=true --from-beginning

```

### Kakfa Config 설정

```
// broker 0번의 config 설정 확인
kafka-configs --bootstrap-server localhost:9092 --entity-type brokers --entity-name 0 --all --describe

// topic의 config 설정 확인
kafka-configs --bootstrap-server localhost:9092 --entity-type topics --entity-name test-topic --all --describe

// topic의 config 설정 변경
kafka-configs --bootstrap-server localhost:9092 --entity-type topics --entity-name test-topic --alter \
--add-config max.message.bytes=2088000

// 변경한 topic의 config를 다시 Default값으로 원복
kafka-configs --bootstrap-server localhost:9092 --entity-type topics --entity-name test-topic --alter \
--delete-config max.message.bytes

// kafka-dump-log 명령어로 log 파일 내부 보기
kafka-dump-log --deep-iteration --files /home/min/data/kafka-logs/multipart-topic-0/00000000000000000000.log --print-data-log
```
