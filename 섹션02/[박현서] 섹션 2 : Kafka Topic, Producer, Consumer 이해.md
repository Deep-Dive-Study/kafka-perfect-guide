# Kakfa Topic, Producer, Consumer 이해 및 CLI로 실습 해보기

- Kafka Topic
  - 병렬 성능과 가용성의 핵심 요소
  - Partition으로 구성된 (일련의) 로그 파일
  - Topic 내에 N개 Partition이 있을 수 있다.
- Kafka Partition
  - sorted, immutable한 일련의 로그 메시지
  - 다른 Kafka Partition과 독립적
- Kafka Record
  - 일련 번호(offset) 할당
- Kafka Broker의 Topic 데이터 복제(replication)
  - Topic의 Partition은 N개의 Kafka Broker에 분산 저장될 수 있다.
  - Kafka Broker 간의 데이터를 각 Topic의 Partition을 Leader와 Follower로 나누는 전략으로 보장한다.
- Kafka Cluster
  - N개의 Kafka Broker를 묶어 Kafka Cluster라고 한다.
- Kafka Producer
  - Topic에 메세지를 송신한다.
  - 성능/로드밸런싱/가용성/업무 정합성 등을 고려하여 어떤 Kafka Broker의 Partition으로 메시지를 송신할지 결정해야 한다.
- Kafka Consumer
  - Topic에서 메시지를 수신한다.
  - N개의 Consumer로 구성될 경우 어떤 Kafka Broker 내에 Partition에서 메시지를 수신할지 결정해야 한다.
- Kafka Consumer Group
  - Consumer는 단 하나의 Consumer Group에 소속되어야 한다.
  - Consumer Group 내에 Consumer에 변화가 있을 경우 Partition과 Consumer의 조합을 변경하는 Rebalancing 진행
  - 단일 Kafka Topic을 Subscribe한 Consumer Group이 N개 있을 경우에 Consumer Group들은 각각 분리되어 독립적이다.
    - ex. 회원 탈퇴 메시지 Produce -> A 서비스 메시지 Consume, B 서비스 메시지 Consume, …
- Kafka Config
  - Broker, Topic Config
    - Static Config
      - Broker 재기동 필요
    - Dynamic Config
      - kafka-configs 이용하여 동적 대응
  - Producer, Consumer Config
    - Client 수행시마다 설정 가능


### 문득 생각한거 끄적이기
  - Key에 따른 Partition 분배 법칙
    - 문제 상황
      - Key 설계가 잘못되었을 때 한 쪽 Partition에 메시지가 몰릴 수 있다.
    - Q. 몰릴 경우에는 무슨 문제가 발생할 수 있는가?
    - A. 간단하게는 한 쪽 Consumer에 메시지 Consume이 몰리겠고 나머지 Consumer는 쉬고 있을 듯.
    - Q. 어차피 Async라면 Consumer가 하나만 돌고 있다고해서 성능 차이가 별로 없지 않는가?
    - A. 무한정으로 스레드를 이용할 수 없으니 스레드 풀을 이용하겠고 그럴 경우라면 다른 Consumer들이 유휴 상태로 있을 것이다.
    - Q. 그러면 어떻게 안몰리게 할 것 인가.
    - A. Kafka Producer 측에서 Key 생성 전략을 고민해봐야한다. 특정 Key 값이 고정되게 한 Partition에 가도록 할 수는 없을거 같다. 파티션 증가/감소는 언제든지 일어날 수 있을거고.. Key값에 대한 Partition이 어떻게 정해지는지 파악하고 Key값이 퍼질 수 있도록 계산해야 한다. (-> murmur2 해싱 알고리즘을 이용한다고 함. 커스텀하면 Partition이 N개 있어도 각각에 Consumer에 Consume할 때 순서 보장까지 할 수 있을 거 같은데.. 그러려면 Partition에 메시지가 저장되고 ack를 받는거 까지 동기로 진행해야함, 아니면 Consumer 측에서 받아서 시간 순서에 맞게 정렬해서 처리해야하는데 Cosnumer도 N개 있을 수 있음 -> Partition N개, Consumer N개여도 시간을 정렬할 수 있는 Key 설계를 하면 될 듯)
