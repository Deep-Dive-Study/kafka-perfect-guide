![image](https://github.com/user-attachments/assets/184b07d0-ef4a-48b3-98a8-744d8e6f09d9)

---

## Java 기반 Producer 구현 실습 및 Producer 내부 메커니즘 이해 - 01

### KafkaProducer

KafkaProducer는 메시지 생성 및 Kafka(Broker)로 전송하는 역할을 하고있다.

### KafkaProducer 설정

- ProducerConfig: KafkaProducer 설정 key 값 정의
- ProducerRecord: KafkaProducer가 Kafka(Broker)에 전달할 메시지 정의

```java
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;

class Main {
    public static void main(String[] args) {
        // 설정
        Properties props = new Properties();
        props.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092"); // 필수
        props.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName()); // 필수
        props.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName()); // 필수

        // KafkaProducer 생성
        KafkaProducer<String, String> kafkaProducer = new KafkaProducer<>(props);       
    }
}
```

<br/>
<br/>

### KafkaProducer 구성

KafkaProducer 내부는 크게 3개로 볼 수 있다.

1. Client가 사용하는 KafkaProducer
1. Client가 send 호출 시 Record가 저장될 RecordAccumulator
1. RecordAccumulator에 저장된 Record를 Broker에 전송할 Sender

![image](https://github.com/user-attachments/assets/6ca9fd48-6ea6-4eb3-9aeb-8fb07260add2)

- KafkaProducer.send(...) 호출 시
    - RecordBatch를 RecordAccumulator에 append
    - RecordAccumulator 내에 해당하는 Topic의 Partition의 Batch에 쌓인다.
    - Sender는 RecordAccumulator 내에 가득 차거나 특정 시간이 지난 RecordBatch를 가져가서 Kafka(Broker)에 전송한다.
- RecordAccumulator
  - Map<TopicPartition, Deque<RecordBatch> 구조로 되어 있다. 
  - Topic의 Partition 마다 Batch를 담고 있는 Deque가 있고 Batch 내에 메시지가 있다.
- Sender
    - KafkaProducer는 별도의 Sender로 RecordAccumulator에 저장된 Batch를 Kafka(Broker)에 전송한다.

<br/>
<br/>

![image](https://github.com/user-attachments/assets/8ba1643e-86a7-4870-b82d-bb1f32e8d6e6)

Batch에 메시지를 담기 위해서는 아래 3가지 작업이 진행된다.

Serialization, Partitioning, Compression이 수행된 후 RecordAccumulator의 배치에 저장된다.

- Serialization
  - Record의 Key, Value를 설정한 Serializer를 이용해 ByteArray로 변환
- Partitioning
  - Partitioner는 Record를 받아 Partition Number를 반환
- Compression
  - Record 압축 (압축 시 사용할 코덱 지정이 가능하다. `default: none`)

<br/>
<br/>

### Producer 동기/비동기 전송 코드

KafkaProducer로 동기/비동기 처리가 가능하다.

실제로 KafkaProducer.send(...) 호출 후에 메시지 전송은 비동기로 수행되지만 Kafka(Broker)의 ack 응답을 동기/비동기로 처리할 수 있다.

Producer의 동기/비동기 전송에 대한 차이는 ack 응답 처리의 차이다. (java.util.concurrent.Future)

```java
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;
import org.apache.kafka.clients.producer.internals.FutureRecordMetadata;
import java.util.Properties;

class Main {
    public static void main(String[] args) {
        Properties props = new Properties();
        KafkaProducer<String, String> kafkaProducer = new KafkaProducer<>(props);
        ProducerRecord<String, String> producerRecord = new ProducerRecord<>(topicName, "hello world");

        // 동기 전송
        RecordMetadata sync = kafkaProducer.send(producerRecord).get(); // Future.get()

        // 비동기 전송 (메시지 전송 성공 확인 X)
        FutureRecordMetadata async = kafkaProducer.send(producerRecord); // Callback이 null로 들어갈 경우 내부에서 예외를 그냥 삼킨다.

        // 비동기 전송 (메시지 전송 성공 확인 O)
        FutureRecordMetadata asyncWithCallback = kafkaProducer.send(producerRecord, callback);        
    }
}
```

- 동기 전송
  - `kafkaProducer.send(producerRecord).get()`
  - Future를 get()하여 ack 응답을 기다림으로써 동기로 볼 수 있다.
- 비동기 전송 (메시지 전송 성공 확인 X)
  - Callback 인자가 null로 들어갈 경우 내부에서 예외를 삼킨다.
- 비동기 전송 (메시지 전송 성공 확인 O)
  - 구현된 Callback 인자를 넣을 경우에 ack 응답을 비동기 처리할 수 있다.

<br/>
<br/>

### Producer 비동기 전송 시 Callback 처리

전송 후 Callback으로 ack 응답을 비동기 처리할 수 있다.

exception 또는 RecordMetadata를 받아 완료처리 할 수 있다.

```java
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.Callback;
import java.util.Properties;

class Main {
    public static void main(String[] args) {
        Properties props = new Properties();
        KafkaProducer<String, String> kafkaProducer = new KafkaProducer<>(props);
        
        for (int seq = 0; seq < 20; seq++) {
            ProducerRecord<String, String> producerRecord = new ProducerRecord<>(topicName, "seq-" + seq, "hello world " + seq);
            CustomCallback callback = new CustomCallback(seq); // 커스텀 콜백
            kafkaProducer.send(producerRecord, callback); // 비동기 전송
        }
    }
}

class CustomCallback implements Callback {
    private int seq;

    public CustomCallback(final int seq) {
        this.seq = seq;
    }

    // 완료처리
    @Override
    public void onCompletion(final RecordMetadata metadata, final Exception exception) {
        if (exception == null) {
            logger.info("seq:{} partition:{} offset:{}", this.seq, metadata.partition(), metadata.offset());
        } else {
            logger.error("exception error from broker " + exception.getMessage());
        }
    }
}
```

<br/>
<br/>

### Producer Key

- ProducerRecord에 key값을 설정해줄 경우 특정 Partition으로 고정되어 전송할 수 있다.

```java
import java.util.Properties;

class Main {
    public static void main(String[] args) {
        Properties props = new Properties();
        KafkaProducer<String, String> kafkaProducer = new KafkaProducer<>(props);

        // Key (X)
        ProducerRecord<String, String> recordWithoutKey = new ProducerRecord<>(
                topicName,    // topic
                "hello world" // value
        );
        kafkaProducer.send(recordWithoutKey);

        // Key (O)
        for (int seq = 0; seq < 20; seq++) {
            ProducerRecord<String, String> recordWithKey = new ProducerRecord<>(
                    topicName,           // topic
                    "seq-" + seq,        // key
                    "hello world " + seq // value
            );
            kafkaProducer.send(recordWithKey);
        }
    }
}
```

(key 유무에 대한 내용은 이전 섹션에서 다뤘다. 여기서는 처리 과정만 보자.)


**Key에 대한 RecordAccumulator 처리 과정**

![image](https://github.com/user-attachments/assets/93eb6a42-a466-4ec6-88e3-828450c1152b)

**💬 Client가 KafkaProducer.send(...) 호출 시에 Record를 RecordAccumulator에 저장할 때 내부에서 어떻게 처리될까?**

RecordAccumulator는 Map 구조로 `key:value=TopicPartition:Deque<RecordBatch>`로 구성되어 있다.

key값은 Record 내부의 topic 값 그리고 Partitioner가 처리하여 반환하는 Partition Number를 이용하여 알아낸다.

```java
package org.apache.kafka.common;

import java.io.Serializable;
import java.util.Objects;

/**
 * A topic name and partition number
 */
public final class TopicPartition implements Serializable {
    private static final long serialVersionUID = -613627415771699627L;

    private int hash = 0;
    private final int partition;
    private final String topic;

    // ...
}
```

**💬 Deque?**

RecordAccumulator에 Queue가 아닌 Deque를 사용한 이유는 가장 최근 생성된 RecordBatch에 Record를 저장할 공간이 있는지 확인하기 위해서이다.

![image](https://github.com/user-attachments/assets/60137fa8-4ef4-484d-a6b5-f6678e9d3931)

**💬 RecordBatch는 언제 새롭게 만들어질까?**

Deque에서 last를 확인 했을 때 수용 불가능하면 BufferPool에서 RecordBatch를 하나 할당 받아 Deque에 append한다.

![image](https://github.com/user-attachments/assets/9980cfa2-3718-49df-ae82-7dc062fe17b4)

<br/>
<br/>

## Java 기반 Producer 구현 실습 및 Producer 내부 메커니즘 이해 - 02

### Producer의 acks 전략

여기서 acks는 Producer가 메시지 전송 후 Kafka의 메시지 수신 성공에 대한 확인/미확인 전략이다.

| OPTION     | Message LOSS | SPEED | DESCRIPTION                                     |
|------------|--------------|-------|-------------------------------------------------|
| acks = 0   | 상            | 상     | Kafka(Broker) 수신 성공을 확인하지 않는다.                  |
| acks = 1   | 중            | 중     | Kafka(Broker) leader의 수신 성공만 확인한다.              |
| acks = all | 하            | 하     | Kafka(Broker) leader, follower의 수신 성공을 모두 확인한다. |


`acks=0`

Producer가 Broker의 leader에게 메시지를 전송하고 수신 성공은 확인하지 않는다.

메시지 일부 손실에 문제가 없고 전송 속도가 중요할 때 선택하는 전략이다. 

![image](https://github.com/user-attachments/assets/e8640dde-f435-44fb-b4e1-9e089fda5183)

`acks=1`

Producer가 Broker의 leader에게 메시지를 전송하고 수신 성공을 확인한다.

acks=0 전략보다 메시지 손실 위험은 줄었으나 수신 성공에 대한 응답을 기다려야하는 시간이 추가되어 전송 속도가 느려진다.

![image](https://github.com/user-attachments/assets/325a2e73-fb91-4deb-a76b-0cdf1c0fe6a1)

메시지 손실 위험이 완전하게 제거되는 전략은 아니다.

leader의 수신 성공만 확인하고 follower의 복제 성공은 확인하지 않기 때문이다.

아래 예시로 leader가 수신 성공을 Producer에 응답한 상태에서 follower 복제 전에 leader에 문제가 생긴다고 해보자. 이때 follower가 leader로 승격되는데 producer 입장에서는 그 사실을 모르고 다음 메시지를 전송하게 된다. 

![image](https://github.com/user-attachments/assets/6a109611-1607-46f7-a0f5-ab8555b9f1de)

`acks=all`

acks=1에서는 leader의 수신 성공만 확인했다면 acks=all에서는 follower의 복제 성공까지 확인한다.

![image](https://github.com/user-attachments/assets/f943e1c8-dca9-46de-a210-54f6a53e6664)

**💬 여기서 follower가 2개라고 했을 때 1개만 복제 성공하면 어떻게 될까?**

지금까지의 정보만으로는 producer에게 어떤 응답을 할지 알 수 없다.

그래서 write 성공하기 위한 최소 복제본의 수를 정해줘야 한다.

broker의 옵션(`min.insync.replicas`)으로 최소 복제본의 수를 지정하면된다. 

![image](https://github.com/user-attachments/assets/5afa193e-f1e6-4080-890f-36dd4c0b14e1)

<br/>
<br/>

### 메시지 배치 전송 내부 메커니즘 - Record Batch와 Record Accumulator

KafkaProducer.send(...) 호출 시마다 Record를 RecordAccumulator(Producer Client)에 저장한다.

RecordAccumulator에 저장된 메시지들은 Topic의 Partition에 따라서 RecordBatch 단위로 묶여 전송된다.

RecordAccmulator에 보관될 수 있는 사이즈는 `buffer.memory`(전체 메모리 사이즈)로 설정할 수 있다.

<br/>
<br/>

### 메시지 배치 전송 내부 메커니즘 - linger.ms와 batch.size

- `batch.size`
  - 단일 Batch의 사이즈
- `linger.ms`
  - Sender가 Batch를 가져갈 때 대기하는 시간
- `max.inflight.request.per.connection`
  - Sender가 Kafka(Broker)에 보낼 Batch 개수

<br/>
<br/>

### Producer 동기/비동기 배치 전송

Kafka Producer는 여러 메시지를 하나의 Batch로 묶어 N개의 Batch를 전송할 수 있어 성능이 좋다.

다만 동기 배치 전송의 경우에는 성능을 포기하고 메시지의 정확한 전송 여부를 선택하는 경우이다.

- 동기 배치 전송
  - 메시지 배치 처리 불가
    - 개별 메시지 별로 응답(ack) 받을 때 까지 block 되기 때문이다.
- 비동기 배치 전송
  - 메시지 배치 처리 가능

<br/>
<br/>

### Producer 재전송 메커니즘

KafkaProducer는 `retries` 값 설정 시에 자동으로 재시도한다.

보통 전송 재시도 횟수는 max 값으로 두고 재시도 간격(retry.backoff.ms)과 최대 시간(delivery.timeout.ms)을 설정해서 처리한다.

```java
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;

class Main {
  public static void main(String[] args) {
    String topicName = "example-topic";
    Properties props = new Properties();
    props.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    props.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
    props.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

    // acks setting
     props.setProperty(ProducerConfig.ACKS_CONFIG, "1");
//     props.setProperty(ProducerConfig.ACKS_CONFIG, "all");

    // linger (ms)
    props.setProperty(ProducerConfig.LINGER_MS_CONFIG, "20");
    
    // retry backoff (ms)
    props.setProperty(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, "1000");
    
    // delivery timeout (ms)
    props.setProperty(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, "5000");

    KafkaProducer<String, String> kafkaProducer = new KafkaProducer<>(props);


    /**
     * 전송 실패 시에 1,000ms 간격으로 최대 5,000ms까지 재시도한다.
     */
    kafkaProducer.send(...);
  }
}
```

- `max.block.ms`
  - send() 호출 시 RecordAccumulator에 입력하지 못하고 block되는 최대 시간
  - 초과 시 TimeoutException
- `request.timeout.ms`
  - 전송에 걸리는 최대 시간
  - 초과 시 retry 또는 TimeoutException
- `retries`
  - 전송 재시도 횟수 (default: Integer.MAX_VALUE)
- `retry.backoff.ms`
  - 전송 재시도 대기 시간
- `deliver.timeout.ms`
  - deliver.timeout.ms >= linger.ms + request.timeout.ms + retry.backoff.ms * N번
  - Producer 메시지 배치 전송 허용된 최대 시간
  - 초과 시 TimeoutException

<br/>
<br/>

### at most once, at least once, idempotence

- `at most once`
  - `acks=0`
  - 적어도 한 번 전송
  - Kafka(Broker)의 ack 응답과 상관없이 Producer가 1번만 전송한다.
  - 장점: 중복 전송 X
  - 단점: 메시지 소실 가능
- `at least once`
  - `acks=1`, `acks=all`
  - `retries > 0`
  - 최소 한 번 전송
  - Kafka(Broker)에 메시지가 정상적으로 기록됐지만 ack에서 문제가 됐을 경우 Producer는 중복 전송이 가능하다.
  - 장점: 메시지 소실 가능
  - 단점: 중복 전송 가능
- `idempotence`
  - 중복 없이 전송
  - Kafka(Broker)에 배치 전송 시에 Header에 producer ID와 sequence를 저장하여 전송한다.
    - sequence는 producer ID마다 고유하다.
  - retry 시에 중복 저장을 피한다.
    - Producer가 배치 전송 재시도 시에 Kafka(Broker)는 seq 값을 확인하여 중복 저장을 피한다.
  - retry가 아닐 경우 중복 저장 가능성이 있다.
    - 배치 전송 재시도가 아닌 Producer 서버 재시작 시에 중복 저장 가능성이 생긴다.
    - Producer ID가 새롭게 바뀌어서 Kafka(Broker) 측에서 seq도 새롭게 받아들인다. 따라서 중복 처리가 불가하다.
    - 이를 해결하기 위해서는 transactional.id를 설정하면된다. Producer 서버가 재시작돼도 기존 Producer ID를 유지하여 중복 처리를 방지할 수 있다.

![image](https://github.com/user-attachments/assets/9da1f33a-8161-4dd6-8144-ccfa44a14da5)

<br/>
<br/>

### Producer Partitioner

- Key가 없는 메시지
  - 라운드 로빈 방식으로 partition 결정
- Key가 있는 메시지
  - Kafka Producer는 murmur2 해시 함수를 사용해서 partition 결정
- 커스텀
  - Partitioner 인터페이스를 구현하여 사용자 정의 가능

```java
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;
import org.apache.kafka.clients.producer.Partitioner;

class CustomPartitionerProducer {
    public static void main(String[] args) {
        String topicName = "custom-topic-partitioner";

        Properties props = new Properties();
        props.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        
        // 커스텀 partitioner 등록
        props.setProperty(ProducerConfig.PARTITIONER_CLASS_CONFIG, CustomPartitioner.class.getName());
    }
}

// 커스텀 partitioner
class CustomPartitioner implements Partitioner {
    @Override
    public void configure(final Map<String, ?> configs) {
        // ...
    }

    @Override
    public int partition(final String topic, final Object key, final byte[] keyBytes, final Object value, final byte[] valueBytes, final Cluster cluster) {
        // ...
    }

    @Override
    public void close() {
        // ...
    }
}
```

<br/>
<br/>

> 참고
> - https://d2.naver.com/helloworld/6560422
> - https://www.popit.kr/kafka-%EC%9A%B4%EC%98%81%EC%9E%90%EA%B0%80-%EB%A7%90%ED%95%98%EB%8A%94-producer-acks/
