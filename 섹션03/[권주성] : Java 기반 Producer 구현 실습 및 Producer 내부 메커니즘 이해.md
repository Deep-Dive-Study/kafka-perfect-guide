# 섹션 3-4 Java 기반 Producer 구현 실습 및 Producer 내부 메커니즘 이해

## Kafka Producer 란?
- Kafka Producer란, **`보통 Kafka Producer Client API를 통해 구현된 애플리케이션을 의미`** 함
  - Kafka Broker에 특정 Topic(혹은 파티션 영역까지)으로 메시지(데이터)를 전달함
  - 즉, 데이터 소스(원본 데이터)에서 Broker로 데이터를 전송을 담당하는 역할을 Producer라고 함

### Kafka Producer Client Overview

![1_JlV07htJU-7m_Yc-cOgjMA](https://github.com/user-attachments/assets/65fcf4fe-88fe-4d39-968f-cdf5e9afd7d1)

### 구성 요소
- `Producer Record`
  - Producer를 통해 전달되는 메시지 객체
  - 구성요소
    - 토픽 (Topic)
    - 토픽 중 특정 파티션 위치 (Partition)
    - 메시지 키 (Key)
    - 메시지 값 (Value)
    - 메시지 생성 시간 (Timestamp)

- `Producer`
  - Client 객체
  - send 메서드 호출을 통해 데이터 전송을 명령할 수 있음
  - 내부적으로 Serialize, Partition, Compression 등의 작업이 수행됨
  - 데이터 전달 완료 메시지를 Future 객체를 통해 응답 받음
 
- `Record Accumulator`
  - Record 가 전송되기 전 저장되어지는 Buffer
  - 어떤 Topic/Partition으로 전송되는지에 따라 나눠서 분류
  - Record는 Batch 단위로 묶여서 저장

- `Sender`
  - Batch 단위로 Broker에 Record 객체를 전송하는 역할
  - 별도의 스레드로 동작

![kafka-simpleProducer](https://github.com/user-attachments/assets/7b0bcc6a-08ee-4bfa-9a6f-79e736b11b05)

### Kafka Producer Client Send Workflow
- Kafka Producer에 Send 메서드를 호출하게 되면 아래 과정을 진행하여 데이터를 전송함
  - 직렬화 (Serializer)
    - Record의 Key, Value 타입에 맞게 지정한 Serializer를 통해 ByteArray 형태로 변환

  - 파티셔닝 (Partitioner)
    - 직렬화 과정을 마친 Record는 Partitioner를 통해 Topic의 어떤 파티션에 저장될지 결정됨
    - Partitioner는 정의된 로직에 따라 파티셔닝을 진행하는데, 별도의 Partitioner 설정을 하지 않으면 Round Robbin 형태로 파티셔닝됨
      - 정확히는 Key가 있는 경우 / 없는 경우에 따라 기본 전략이 다름 (참고: )

  - 압축 (Compression)
    - compression.type을 설정하여 Record는 Record Accumulator에 저장할 때 압축하여 저장할 수 있음 (기본값은 none)
    - 압축을 통해 네트워크 전송 비용과 저장 비용도 줄일 수 있음

  - 배치 저장 (Record Accumulator)
    - 어떤 파티션으로 전송되는지에 따라 Batch 단위로 데이터가 일시적으로 저장되는 공간
    - Kafka는 효율적인 데이터 전송을 위해 데이터를 즉시 전송하지 않고 설정된 시간이나 데이터 크기가 충족되기 전까지 데이터를 전송하지 않고 Batch 단위로 데이터를 한번에 전송함 
    - 따라서, Record는 전송 전에 먼저 Record Accumulator에 Batch 단위로 묶여서 저장됨

  - 전송 (Sender)
    - 별도의 스레드로 동작하여 비동기로 데이터를 전송
    - 설정된 시간 혹은 사이즈가 충족되어야 데이터를 전송함
 
## 실습
### KafkaProducer 설정

```java
public class Main {
    public static void main(String[] args) {
        // 먼저 설정 객체를 통해 초기화 작업 진행
        Properties props = new Properties();
        props.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092"); // 연결할 Broker 서버 - 하나만 지정해도 나머지 정보를 가져옴. 다만, 지정한 서버가 다운될 경우를 고려하여 두개 정도 지정하는 것을 권장
        props.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName()); // 키 직렬화 지정
        props.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName()); // 값 직렬화 지정

        // KafkaProducer 생성
        KafkaProducer<String, String> kafkaProducer = new KafkaProducer<>(props);       
    }
}
```

### 데이터 전송

```java
public class Main {
    public static void main(String[] args) {
        // 먼저 설정 객체를 통해 초기화 작업 진행
        Properties props = new Properties();
        props.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

        // KafkaProducer 생성
        KafkaProducer<String, String> kafkaProducer = new KafkaProducer<>(props);

        // 메시지(Record) 생성
        ProducerRecord<String, String> producerRecord = new ProducerRecord<>("topic-name", "key", "message"); // key는 생략 가능(파티션 지정용)

        // 동기 전송
        RecordMetadata sync = kafkaProducer.send(producerRecord).get(); // Future 객체를 get()으로 받을 경우 동기식으로 동작함(즉, 데이터가 올때까지 기다림)

        // 비동기 전송
        FutureRecordMetadata async = kafkaProducer.send(producerRecord); // 별도로 메시지를 확인하지 않음

        // 비동기 전송 - 콜백
        FutureRecordMetadata asyncWithCallback = kafkaProducer.send(producerRecord, callback); // 콜백을 통해 메시지/예외 확인    
    }
}
```

### 동기(Sync) 전송 vs 비동기(Async) 전송
- `동기식 전송`
  - Producer는 브로커로 부터 해당 메시지를 성공적 으로 받았다는 Ack 메시지를 받은 후 다음 메시지를 전송하는 방식
  - KafkaProducer.send( ).get( ) 호출하여 브로커로 부터 Ack 메시지를 받을 때까지 대기(Wait)

- `비동기식 전송`
  - Producer는 브로커로 부터 해당 메시지를 성공적으로 받았다는 Ack 메시지를 기다리지 않고 전송하는 방식
  - 브로커로 부터 Ack 메시지를 비동기로 Producer에 받기 위해서 Callback을 적용해야함
  - send( ) 메소드 호출 시에 callback 객체를 인자로 입력하여 Ack 메시지의 MetaData(혹은 Exception)를 Producer로 전달 받을 수 있음

![CleanShot 2025-02-01 at 21 41 34@2x](https://github.com/user-attachments/assets/4aa61749-c28f-49d8-a733-19733e9cd858)


### Producer의 acks 설정에 따른 send 방식
- Producer의 acks 방식에 따라 Producer가 메시지 전송 후에 메시지 수신 성공에 대한 확인을 어느 범위까지 할 것인지 설정할 수 있음
  - 이는 전송 속도와 정확도 간의 트레이드 오프 관계에 대한 설정
  - 다만, Cusumer가 데이터를 소비하는 속도와는 무관함(Cusumer는 데이터 복제가 끝나고 나서 소비 가능)

#### acks = 0 인 경우 
- Ack 메시지를 받지 않고 다음 메시지인 메시지 B를 바로 전송
  - Producer는 Leader Broker가 메시지 A를 정상적으로 받았는지 확인하지 않음
  - 메시지가 제대로 전송되었는지 브로커로 부터 확인을 받지 않기 때문에 메시지가 브로커에 기록되지 않더라도 재 전송을 시도하지 않음

- 메시지 손실의 우려가 가장 크지만 가장 빠르게 전송할 수 있음
  - IOT 센서 데이터등 데이터의 손실에 민감하지 않은 데이터 전송에 활용함

  ![CleanShot 2025-02-01 at 22 28 12@2x](https://github.com/user-attachments/assets/b4fb8164-2aee-4177-ad76-646f4d4f41a8)


#### acks = 1 인 경우 
- Producer는 Leader Broker가 메시지 A를 정상적으로 받았는지에 대한 Ack 메시지를 받은 후 다음 메시지인 메시지 B를 바로 전송
  - 만약 오류 메시지를 브로커로 부터 받으면 메시지 A를 재 전송함
  - 메시지 A가 모든 Replicator(follower)에 완벽하게 복사되었는지의 여부는 확인하지 않음

- 만약, Leader가 메시지를 복제 중에 다운될 경우 다음 Leader가 될 브로커에는 메시지가 없을 수 있기 때문에 메시지를 소실할 우려가 약간 있음

  ![CleanShot 2025-02-01 at 22 28 24@2x](https://github.com/user-attachments/assets/d3ac5f29-aefc-4a2a-9ee6-f42112301837)

#### acks = all 인 경우 
- Producer는 Leader Broker가 메시지 A를 정상적으로 받은 뒤 min.insync.replicas 개수 만큼의 Replicator(follower)에 복제를 수행한 뒤에 보내는 Ack 메시지를 받은 후 다음 메시지인 메시지 B를 바로 전송
  - 만약 오류 메시지를 브로커로 부터 받으면 메시지 A를 재 전송함
  - 메시지 A가 모든 Follower에 완벽하게 복사되었는지의 여부까지 확인한 후에야 메시지 B를 전송함

- 메시지 손실이 되지 않도록 모든 장애 상황을 감안한 전송 모드이지만 Ack를 오래 기다려야 하므로 상대적으로 전송속도가 느림.

  ![CleanShot 2025-02-01 at 22 28 36@2x](https://github.com/user-attachments/assets/46e72228-20ce-4593-9c45-2bda7659e39b)

#### Producer의 Sync와 Callback Async에서의 acks와 retry
- Callback 기반의 async에서도 동일하게 acks 설정에 기반하여 retry가 수행됨
- Callback 기반의 async에서는 retry에 따라 Producer의 원래 메시지 전송 순서와 Broker에 기록되는 메시지 전송 순서가 변경 될 수 있음
  - 이를 해결하기 위해서는 멱등성 설정(**`enable.idempotence = true`**)이 필요함
- Sync 방식에서 acks=0일 경우 전송 후 ack/error를 기다리지 않음(fire and forget)


## Batch 전송 및 Retry 동작

### Batch 전송
  ![CleanShot 2025-02-01 at 22 27 58@2x](https://github.com/user-attachments/assets/d10f95dd-bdea-4643-9306-c4a36155f5cd)

- Producer의 send 메소드는 호출 시마다 하나의 Producer Record를 입력받지만 곧 바로 전송 되지 않고 내부 메모리에서 단일 메시지를 토픽 파티션에 따라 Record Batch 단위로 묶인 뒤 전송됨
- 메시지들은 Producer Client의 내부 메모리에 여러 개의 Batch들로 **`buffer.memory`** 설정 사이즈 만큼 보관될 수 있으며 여러 개의 Batch들로 한꺼번에 전송될 수 있음
- Record Accumulator는 Partitioner에 의해서 메시지 배치가 전송이 될 토픽과 Partition에 따라 저장되는 KafkaProducer 메모리 영역

  ![미리보기 2025-02-01 22 30 42](https://github.com/user-attachments/assets/2efbeea2-176a-4866-8987-a5813b128ba4)

- Sender Thread는 Record Accumulator에 누적된 메시지 배치를 꺼내서 브로커로 전송
- Kafka Producer의 Main Thread는 send( ) 메소드를 호출하고 Record Accumulator 에 데이터 저장하고 Sender Thread는 별개로 데이터를 브로커로 전송
- Sender Thread는 기본적으로 전송할 준비가 되어 있으면 Record Accumulator에서 1개의 Batch를 가져갈수도, 여러 개의 Batch를 가져 갈 수도 있음. 또한 Batch에 메시지가 다 차지 않아도 가져 갈 수 있음
- **`linger.ms`** 를 0보다 크게 설정하여 Sender Thread가 하나의 Record Batch를 가져갈 때 일정 시간 대기하여 Record Batch에 메시지를 보다 많이 채울 수 있도록 적용

  ![CleanShot 2025-02-01 at 22 35 55@2x](https://github.com/user-attachments/assets/f6a8d29a-bf69-4aa1-a877-0321be865b9f)

- linger.ms를 반드시 0 보다 크게 설정할 필요는 없음
- Producer와 Broker간의 전송이 매우 빠르고 Producer에서 메시지를 적절한 Record Accumulator에 누적된다면 linger.ms가 0이 되어도 무방
- 전반적인 Producer와 Broker간의 전송이 느리다면 linger.ms를 높여서 메시지가 배치로 적용될 수 있는 확률을 높이는 시도를 해볼 만함.
- linger.ms는 보통 20ms 이하로 설정 권장

#### Producer의 Sync와 Callback Async에서의 Batch
- Callback 기반의 Async는 비동기적으로 메시지를 보내면서 Record Metadata를 Client가 받을 수 있는 방식을 제공
  - Callback 기반의 Async는 여러 개의 메시지가 Batch로 만들어짐.
- `RecordMetaData recordMetadata = KafkaProducer.send().get()` 와 같은 방식으로 개별 메시지 별로 응답을 받을 때까지 block이 되는 방식으로는 메시지 배치 처리가 불가함
  - 전송은 배치 레벨이지만 배치에 메시지는 단 1개

#### Producer의 메시지 전송/재 전송 시간 파라미터 이해
- `delivery.timeout.ms` >= `linger.ms` + `request.timeout.ms`
  - 해당 설정이 지켜지지 않는 경우에는 어플리케이션이 동작하지 않음

  ![CleanShot 2025-02-01 at 22 37 46@2x](https://github.com/user-attachments/assets/90582084-3dc6-464c-9a0f-ba6bb8547ca0)

  ![CleanShot 2025-02-01 at 21 49 55@2x](https://github.com/user-attachments/assets/98e22c63-5eb7-494c-93ca-b9f4f1c0800d)
  
### Retry 동작

![미리보기 2025-02-01 22 48 01](https://github.com/user-attachments/assets/6f17a913-60d4-468d-9da4-a9c3997aeb93)

- `retries`와 `delivery.timeout.ms` 를 이용하여 재 전송 횟수 조정할 수 있음
  - `retries`는 재 전송 횟수 설정
  - `delivery.timeout.ms` 는 메시지 재 전송을 멈출때 까지의 시간

- 보통 retries는 무한대값으로 설정하고 delivery.timeout.ms(기본 120000, 즉 2분) 를 조정하는 것을 권장함

![미리보기 2025-02-01 22 48 07](https://github.com/user-attachments/assets/2b009408-8671-45dd-8e10-6b4a025b2716)

- retry.backoff.ms는 재 전송 주기 시간을 설정
- retries=10, request.timeout.ms=10000ms, retry.backoff.ms=30인 경우 request.timeout.ms 기다린후 재 전송하기전 30ms 이후 재전송 시도. 이와 같은 방식으로 재 전송을 10회 시도하고 더이상 retry 시도 하지 않음.
- 만약 10회 이내에 delivery.timeout.ms에 도달하면 더 이상 retry 시도하지 않음.

#### max.in.flight.requests.per.connection 이해
- 브로커 서버의 응답없이 Producer의 sender thread가 한번에 보낼 수 있는 메시지 배치의 개수(기본 값은 5)
  - Kafka Producer의 메시지 전송 단위는 Batch임.
- 비동기 전송 시 브로커의 응답없이 한꺼번에 보낼 수 있는 Batch의 개수는 `max.in.flight.requests.per.connection` 에 따름

### 최대 한번 전송, 적어도 한번 전송, 정확히 한번 전송

#### 최대 한번 전송(at most once)
- Producer는 브로커로 부터 Ack 또는 Exception 없이 다음 메시지를 연속적으로 보냄
  - 메시지가 소실 될 수는 있지만 중복 전송은 하지 않음
  
  ![CleanShot 2025-02-01 at 22 53 52@2x](https://github.com/user-attachments/assets/ff5f014d-f74f-4449-8ab5-cdb85cfd0d8d)

#### 적어도 한번 전송(at least once)
- Producer는 브로커로 부터 ACK를 받은 다음에 다음 메시지 전송
  - 메시지 소실은 없지만 중복 전송을 할 수 있음

  ![CleanShot 2025-02-01 at 22 54 05@2x](https://github.com/user-attachments/assets/122f71dc-d5b8-4a50-9598-592d16d292ef)

#### 정확히 한번 전송(exactly once)
- 중복 없는 전송(idempotence)을 의미
- Producer는 브로커로 부터 ACK를 받은 다음에 다음 메시지 전송하되, Producer ID와 메시지 Sequence를 Header에 저장하여 전송
- 메시지 Sequence는 메시지의 고유 Sequence 번호.
  - 0부터 시작하여 순차적으로 증가.
  - Producer ID는 Producer가 기동시마다 새롭게 생성되는 값
- 브로커에서 메시지 Sequece가 중복 될 경우 이를 메시지 로그에 기록하지 않고 Ack만 전송
- 브로커는 Producer가 보낸 메시지의 Sequence가 브로커가 가지고 있는 메시지의 Sequence보다 1만큼 큰 경우에만 브로커에 저장
- Idempotence 적용 후 성능이 약간 감소(최대 20%)할 수 있지만 기본적으로 idempotence 적용을 권장

  ![CleanShot 2025-02-01 at 22 54 31@2x](https://github.com/user-attachments/assets/cd3bd407-bd76-4cc3-851f-477fdc745fff)

### Idempotence 를 위한 Producer 설정
- enable.idempotence = true
- acks = all
- retries는 0 보다 큰 값
- max.in.flight.requests.per.connection은 1에서 5사이 값
- 해당 설정이 지켜지지 않는 경우, 동작하지 않음
  - 다만, Kafka 3.0 버전 부터는 Producer의 기본 설정이 Idempotence가 됨
  - 이경우는 기본 설정중에 enable.idempotence=true를 제외하고 다른 파라미터들을 잘못 설정하면(예를들어 acks=1로 설정) Producer는 정상적으로 메시지를 보내지만 idempotence로는 동작하지 않는 상황이 발생함
- 따라서, 명시적으로 enable.idempotence=true를 설정한 뒤 다른 파라미터들을 잘못 설정하면 Config 오류가 발생하면서 Producer가 기동되지 않도록 하는 것이 좋음
