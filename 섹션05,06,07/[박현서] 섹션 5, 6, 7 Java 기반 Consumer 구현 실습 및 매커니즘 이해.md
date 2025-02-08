## Java 기반 Consumer 구현 실습 및 Consumer 내부 메커니즘 이해 - 01

### Kafka Consumer 개요

- Broker의 Topic 메시지 소비자
- Consumer는 group.id를 갖는 Consumer Group에 속함
- Consumer Group 내에 Consumer들은 Topic Partition 별로 분배됨

### Kafka Consumer 동작 과정 (subscribe, poll, commit)

- `subscribe()`
    - Consumer가 Broker에게 구독 요청
- `poll()`
    - Consumer가 구독한 Topic의 Partition에서 메시지를 주기적으로 가져옴
    - 첫 번째 poll: 메시지가 아닌 Broker의 metadata를 가져와 GroupCoordinator와 연결
    - 두 번째 poll 이후: 메시지를 가져옴
- `commit()`
    - Consumer는 commit을 통해서 Broker의 Topic의 __consumer_offsets에 offset을 기록

```java
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Duration;
import java.util.List;
import java.util.Properties;

public class SimpleConsumer {
    private static final Logger logger = LoggerFactory.getLogger(SimpleConsumer.class);

    public static void main(String[] args) {
        String topicName = "simple-topic";

        Properties props = new Properties();
        props.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.setProperty(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.setProperty(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());

        // consumer group id
        props.setProperty(ConsumerConfig.GROUP_ID_CONFIG, "group_01");

        final KafkaConsumer<String, String> kafkaConsumer = new KafkaConsumer<>(props);

        // topic 구독
        kafkaConsumer.subscribe(List.of(topicName));

        while (true) {
            // 메시지 (N개) poll
            final ConsumerRecords<String, String> consumerRecords = kafkaConsumer.poll(Duration.ofMillis(1000));

            for (final ConsumerRecord<String, String> record : consumerRecords) {
                logger.info("record key:{}, record value:{}, partition:{}", record.key(), record.value(), record.partition());
            }
        }
    }
}
```

### Kafka Consumer 내부 (Fetcher, ConsumerClientNetwork)

**Fetcher**

Fetcher는 Consumer가 Kafka Broker로부터 메시지를 가져오는 역할을 한다.

![image](https://github.com/user-attachments/assets/0ee74feb-2abb-4dbd-bbf1-155a430aa4c5)

- 역할
    - Consumer가 `poll()` 호출 시에 Fetcher가 실행된다.
    - Fetcher는 할당된 Partition 목록을 기반으로 Broker에 Fetch 요청을 보낸다.
    - Broker가 데이터를 반환하면 Fetcher는 내부 queue에 데이터를 저장한다.
    - Consumer는 데이터를 가져와 처리하고, offset을 커밋한다.

| 설정                        | 설명                                 | 기본값   |
|---------------------------|------------------------------------|-------|
| fetch.min.bytes           | 최소한 가져와야하는 데이터 크기 (데이터 부족하면 대기한다.) | 1byte |
| fetch.max.bytes           | 한 번의 Fetch 요청으로 가져올 최대 데이터 크기      | 50mb  |
| fetch.min.wait.ms         | 데이터가 없을 경우 대기하는 최대 시간              | 500ms |
| max.partition.fetch.bytes | 한 개의 Partition에서 가져올 수 있는 최대 데이터 크기 | 1mb   |
| max.poll.records          | Fetcher가 한 번에 가져올 수 있는 레코드 수       | 500   |

**ConsumerClientNetwork**

Kafka Cluster와 네트워크 통신을 담당하는 역할을 한다.

비동기 I/O를 활용하여 Broker와 네트워크 요청을 효율적으로 처리한다.

- 역할
    - Fetcher가 생성한 Fetch 요청을 Broker에 보낸다.
    - offset commit 요청을 처리한다.
    - rebalancing 중에 새로운 파티션 할당 요청을 보낸다.
    - Broker 장애 감지 및 retry 처리한다.

| 설정                      | 설명                      | 기본값  |
|-------------------------|-------------------------|------|
| request.timeout.ms      | 요청이 응답을 기다리는 최대 시간      | 30s  |
| reconnect.backoff.ms    | broker 연결 실패 시 retry 간격 | 50ms |
| metadata.max.age.ms     | 메타데이터 캐시 갱신 주기          | 5m   |
| connections.max.idle.ms | 비활성 연결을 유지하는 최대 시간      | 9m   |

**Heart Beat Thread**

Kafka Consumer는 Consumer Group 유지를 위해 주기적으로 Broker에 heartbeat를 보낸다.

- 역할
  - Consumer가 Kafka와 연결되면 Heartbeat Thread가 실행된다.
  - 주기적으로 Heartbeat 요청을 Broker의 Coordinator에게 보낸다.
  - Coordinator가 정상 응답하면 Consumer는 계속해서 메시지를 처리할 수 있다.
  - Broker는 heartbeat가 일정 시간 동안 수신되지 않으면 Consumer를 비정상 상태로 간주하고 rebalancing을 진행한다. 

### Kafka Consumer __consumer_offsets, auto.offset.reset

**__consumer_offsets**

Kafka Consumer가 읽은 메시지의 위치(offset)를 저장하는 내부 Topic이다.

Kafka Consumer 재시작 시에도 마지막으로 읽은 위치부터 다시 시작할 수 있도록 도와준다.

**Consumer Group offset 저장**

- Kafka Consumer가 특정 Partition에서 마지막으로 읽은 offset 정보를 저장한다.
- Consumer가 정상적으로 재시작할 수 있도록, __consumer_offsets에 기록된 offset을 기반으로 마지막 읽은 위치를 복구한다.

**__consumer_offsets Topic, Partition**

- __consumer_offsets Topic은 기본 50개의 Partition을 갖는다.
- offset 저장 성능을 높이기 위해 Consumer Group ID를 기반으로 특정 Partition에 저장한다.

**__consumer_offsets 저장 정보**

| 필드                | 설명                     |
|-------------------|------------------------|
| Consumer Group ID | Consumer가 속한 그룹명       |
| Topic             | Consumer가 읽고 있는 토픽명    |
| Partition         | Consumer가 읽고 있는 파티션 번호 |
| Offset            | 마지막으로 커밋된 오프셋 번호       |
| Timestamp         | 해당 오프셋이 커밋된 시간         |

## Java 기반 Consumer 구현 실습 및 Consumer 내부 메커니즘 이해 - 02 

### Group Coordinator와 Consumer/Consumer Group

- Group Coordinator
  - Consumer Group을 관리하는 역할
  - Consumer Group 내의 Consumer가 Broker에 최초 접속 요청 시에 Group Coordinator 생성
  - Consumer 추가/제거 시에 Coordinator가 감지하여 Rebalancing을 트리거한다. (Leader Consumer가 Rebalancing 수행)
- Leader Consumer
  - Consumer Group에 가장 빨리 join 요청한 Consumer를 leader로 지정

### Consumer Rebalancing

- Consumer Rebalancing
  - Consumer Group 내에서 Partition 할당이 변경되는 과정을 의미
  - Rebalancing 발생 시에 Consumer Group 내에 Consumer가 처리하는 Partition이 재할당됨
- Rebalancing 발생
  - Consumer Group에 새로운 Consumer 추가
    - Group Coordinator가 기존 Consumer들이 처리 중인 Partition을 나눠서 재할당
  - Consumer Group에 Consumer 제거 (Consumer 종료 또는 장애 발생 시)
    - 해당 Consumer가 맡고 있던 Partition을 다른 Consumer들에게 재할당
  - Kafka Topic의 Partition 개수 변경
    - Consumer Group은 새로운 Partition을 할당

### Consumer Static Group Membership

- Consumer Static Group Membership
  - Consumer가 재시작되거나 일시적으로 연결이 끊겨도 기존 Partition 할당을 유지할 수 있도록 해주는 기능
  - Rebalancing 발생을 최소화하여 성능 저하 방지할 수 있음
    - 캐시, 세션, 상태 데이터 유지해야하는 경우
    - 불필요한 Rebalancing 방지가 필요한 경우

예시 코드

```java
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.util.Properties;

public class ConsumerWakeup {
    public static void main(String[] args) {
        String topicName = "pizza-topic";

        Properties props = new Properties();
        props.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.setProperty(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.setProperty(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.setProperty(ConsumerConfig.GROUP_ID_CONFIG, "group-01-static"); // Consumer Group ID
        props.setProperty(ConsumerConfig.GROUP_INSTANCE_ID_CONFIG, "1");      // 고유한 Static ID (각 Consumer마다 다르게 설정)

        final KafkaConsumer<String, String> kafkaConsumer = new KafkaConsumer<>(props);
        kafkaConsumer.subscribe(List.of(topicName));
    }
}
```
 
Consumer 3개 생성 후 Consumer 1개를 제거한다.

`session.timeout.ms`이 지나면 Rebalancing을 진행한다.

아래는 Rebalancing 진행 후 Consumer Group 상태이다.

```bash
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group group-01-static

GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                            HOST            CLIENT-ID
group-01-static pizza-topic     0          95              95              0               1-2b042582-50dd-4c74-9456-4b5df46bdcd2 /127.0.0.1      consumer-group-01-static-1
group-01-static pizza-topic     1          103             105             2               1-2b042582-50dd-4c74-9456-4b5df46bdcd2 /127.0.0.1      consumer-group-01-static-1
group-01-static pizza-topic     2          57              59              2               2-9ed1e66b-920f-450a-8ae0-0ef53267b13c /127.0.0.1      consumer-group-01-static-2
```

Static Group Membership, Dynamic Group Membership 비교 표

| 비교 항목          | Static Group Membership                | Dynamic Group Membership      |
| -------------- | -------------------------------------- | ----------------------------- |
| **리밸런싱 발생 여부** | Consumer 장애/재시작 시 **리밸런싱 없음**          | Consumer 장애/재시작 시 **리밸런싱 발생** |
| **파티션 유지**     | 기존 파티션을 유지함                            | 새롭게 할당받을 수도 있음                |
| **상태 유지**      | Consumer가 기존 상태를 유지 가능                 | Consumer가 새로운 상태로 초기화될 수 있음   |
| **주요 용도**      | 장기 실행되는 Consumer, **캐시/세션 유지가 중요한 경우** | 일반적인 Consumer 사용              |
| **설정 필요 여부**   | `group.instance.id` 설정 필요              | 별도의 설정 없음                     |

### Heart Beat Thread

- Heart Beat Thread
  - Consumer가 poll()할 때 HeartBeatThread 동작한다.

| 설정 값 | 설명 |
|---------|------------------------------|
| **heartbeat.interval.ms** | Heartbeat가 Group Coordinator로 전송되는 주기 |
| **session.timeout.ms** | Consumer 장애를 감지하는 시간 (Heartbeat가 session.timeout.ms 동안 오지 않으면 Consumer 장애로 간주) |
| **max.poll.interval.ms** | `poll()` 호출 간 최대 허용 시간 (이 시간을 초과하면 Consumer는 장애로 간주되어 리밸런싱 발생) |

### Consumer Rebalance Mode (Eager, Cooperative)

| Mode | 특징                   | 장점                   | 단점 | 사용 추천 |
|------|----------------------|----------------------|------|---------|
| **Eager Rebalance** |      기존 Consumer는 모든 파티션을 해제하고 새로 할당받음 | 빠르게 새로운 할당을 결정       | 전체 파티션 해제 → 메시지 처리 중단 발생 | Consumer 변경이 적을 때 |
| **Cooperative Rebalance** | 일부만 해제하면서 점진적으로 변경| 부드러운 리밸런싱, 메시지 처리 지속 | 즉각적인 할당 변경 어려움 | Consumer 변경이 잦을 때 |

- Eager
  - Consumer 추가/제거 시에 모든 Partition을 해제하고 재할당
  - Consumer가 재시작되기에 순간적으로 메시지 처리가 중단됨
  - Consumer가 많을 수록 Rebalancing 비용이 커짐
- Cooperative
  - Consumer가 일부 Partition만 해제하고 새롭게 할당
  - 기존에 사용하던 Partition 유지가 가능해서 서비스 중단이 없음
  - Rebalancing 비용이 적고, 메시지 처리가 중단되지 않음
  - 

### Consumer의 파티션 할당 전략

- Range
  - Consumer에 Partition을 Topic 별로 묶어서 할당
  - 주의
    - Partition의 개수가 Consumer의 개수로 나누어떨어지지 않으면 Consumer가 몰릴 수 있음
- Round Robin
  - Consumer를 순환하며 하나씩 할당하는 방식
- Sticky
  - 이전 최대한 할당을 유지하면서 Consumer 추가/제거됐을 때 필요한 부분만 변경
- Cooperative Sticky
  - Sticky와 같이 기존 할당을 최대한 유지

## Java 기반 Consumer 구현 실습 및 Consumer 내부 메커니즘 이해 - 03

### Auto Commit

- Auto Commit
  - `auto.enable.commit=true` 설정
  - Consumer는 `auto.commit.interval.ms`마다 Broker에 commit 수행
- commit 시점
  - poll()
  - close()
- 중복 상황 발생 가능
  - Consumer 장애/재기동/Rebalancing 후 Broker에서 이미 읽어온 메시지를 다시 읽을 수 있음 (읽어온 메시지가 commit 되지 않고 종료된 경우)

### Manual Sync/Async Commit

- Sync Commit
  - Consumer.commitSync() 이용
  - poll()로 읽어온 메시지의 마지막 offset을 Broker에 Commit
  - Broker에서 Commit 실패 시 다시 Commit 적용 요청
  - Commit 적용 완료 후에 다시 메시지를 읽음

```java
import org.apache.kafka.clients.consumer.CommitFailedException;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.errors.WakeupException;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Duration;
import java.util.List;
import java.util.Properties;

public class ConsumerCommit {
    private static final Logger logger = LoggerFactory.getLogger(ConsumerCommit.class);

    public static void main(String[] args) {
        String topicName = "pizza-topic";

        Properties props = new Properties();
        props.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.setProperty(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.setProperty(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.setProperty(ConsumerConfig.GROUP_ID_CONFIG, "group_03");
        props.setProperty(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false"); // 동기 commit을 위해 auto commit false 설정

        final KafkaConsumer<String, String> kafkaConsumer = new KafkaConsumer<>(props);
        kafkaConsumer.subscribe(List.of(topicName));

        Thread mainThread = Thread.currentThread();
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
             kafkaConsumer.wakeup();

            try {
                mainThread.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }));

        pollCommitSync(kafkaConsumer);
    }

    private static void pollCommitSync(final KafkaConsumer<String, String> kafkaConsumer) {
        int loopCnt = 0;
        try {
            while (true) {
                final ConsumerRecords<String, String> consumerRecords = kafkaConsumer.poll(Duration.ofMillis(1000));
                for (final ConsumerRecord<String, String> record : consumerRecords) {
                    logger.info("record key:{}, partition:{}, record offset:{}, record value:{}", record.key(), record.partition(), record.offset(), record.value());
                }

                try {
                    kafkaConsumer.commitSync(); // 동기 방식 커밋 요청
                } catch (CommitFailedException e) {
                    logger.error(e.getMessage());
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage());
        } finally {
            kafkaConsumer.close();
        }
    }
}
```

- Async Commit
  - Consumer.commitAsync() 이용
  - Broker에서 Commit 실패해도 다음 메시지 읽어옴
  - Callback을 넘길 수 있음

```java
import org.apache.kafka.clients.consumer.CommitFailedException;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.consumer.OffsetAndMetadata;
import org.apache.kafka.clients.consumer.OffsetCommitCallback;
import org.apache.kafka.common.TopicPartition;
import org.apache.kafka.common.errors.WakeupException;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Duration;
import java.util.List;
import java.util.Map;
import java.util.Properties;

public class ConsumerCommit {
  private static final Logger logger = LoggerFactory.getLogger(ConsumerCommit.class);

  public static void main(String[] args) {
    String topicName = "pizza-topic";

    Properties props = new Properties();
    props.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    props.setProperty(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
    props.setProperty(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
    props.setProperty(ConsumerConfig.GROUP_ID_CONFIG, "group_03");
    props.setProperty(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");

    final KafkaConsumer<String, String> kafkaConsumer = new KafkaConsumer<>(props);
    kafkaConsumer.subscribe(List.of(topicName));

    Thread mainThread = Thread.currentThread();
    Runtime.getRuntime().addShutdownHook(new Thread(() -> {
       kafkaConsumer.wakeup();

      try {
        mainThread.join();
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
    }));

    pollCommitAsync(kafkaConsumer);
  }
  
  private static void pollCommitAsync(final KafkaConsumer<String, String> kafkaConsumer) {
    int loopCnt = 0;
    try {
      while (true) {
        final ConsumerRecords<String, String> consumerRecords = kafkaConsumer.poll(Duration.ofMillis(1000));
        for (final ConsumerRecord<String, String> record : consumerRecords) {
          logger.info("record key:{}, partition:{}, record offset:{}, record value:{}", record.key(), record.partition(), record.offset(), record.value());
        }

        // 비동기 commit 처리
        kafkaConsumer.commitAsync(new OffsetCommitCallback() {
          @Override
          public void onComplete(final Map<TopicPartition, OffsetAndMetadata> offsets, final Exception exception) {
            if (exception != null) {
                // 에러 로깅
              logger.error("offsets {} is not completed, error:{}", offsets, exception);
            }
          }
        });
      }
    } catch (Exception e) {
      logger.error(e.getMessage());
    } finally {
      kafkaConsumer.commitSync();
      kafkaConsumer.close();
    }
  }
}
```

### Topic의 특정 Partition 할당하여 가져오기

- KafkaConsumer.assign() 이용
- Consumer에 N개 Partition이 있는 경우 Topic에 특정 Partition만 할당 가능

```java
import org.apache.kafka.clients.consumer.CommitFailedException;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.consumer.OffsetAndMetadata;
import org.apache.kafka.clients.consumer.OffsetCommitCallback;
import org.apache.kafka.common.TopicPartition;
import org.apache.kafka.common.errors.WakeupException;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Duration;
import java.util.List;
import java.util.Map;
import java.util.Properties;

public class ConsumerPartitionAssign {
    private static final Logger logger = LoggerFactory.getLogger(ConsumerPartitionAssign.class);

    public static void main(String[] args) {
        String topicName = "pizza-topic";

        Properties props = new Properties();
        props.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.setProperty(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.setProperty(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.setProperty(ConsumerConfig.GROUP_ID_CONFIG, "group_pizza_assign_seek");
        props.setProperty(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");

        final KafkaConsumer<String, String> kafkaConsumer = new KafkaConsumer<>(props);
        final TopicPartition topicPartition = new TopicPartition(topicName, 0); // 토픽명, 파티션 번호
      
        // Topic의 Partition 할당
        // kafkaConsumer.assign(List.of(topicPartition));
      
        // Topic의 Partition 할당 + 시작할 offset
        kafkaConsumer.seek(topicPartition, 10);
        
        pollCommitSync(kafkaConsumer);
    }
}
```

> 참고
>
> https://d2.naver.com/helloworld/0974525
