# 섹션12 Producer와 Consumer의 Custom 객체 직렬화 및 역직렬화 적용

## Producer와 Consumer간의 Custom 객체의 전송

![CleanShot 2025-02-22 at 22 00 30@2x](https://github.com/user-attachments/assets/ee26c2f5-81d4-46d0-af22-a8f448c72c16)

- `직렬화(Serialization)`
  - Kafka Producer가 메시지를 전송하기 전에, 객체나 데이터를 네트워크를 통해 전송 가능한 바이트 배열 형태로 변환하는 과정
    - ex) 문자열, JSON, 또는 사용자 정의의 Custom 객체를 Kafka에 전송하려면 이를 적절한 바이트 배열로 변환해야 함

- `역직렬화(Deserialization)`
  - Kafka Consumer가 전송된 메시지를 수신한 후, 바이트 배열로 인코딩된 데이터를 다시 원래의 객체나 원하는 데이터 형식으로 복원하는 과정

- 직렬화와 역직렬화는 동일한 타입(프로토콜)이여야 정확하게 데이터를 주고받을 수 있음
- kafka에서 기본적으로 제공하는 Serializer / Deserializer (일반적인 Primitive Type 에 대해서만 제공)
  - StringSerializer
  - ShortSerializer
  - IntegerSerializer
  - LongSerializer
  - DoubleSerializer
  - BytesSerializer

- 즉, Custom 객체를 전송하기 위해서는 Custom Serializer/Deserializer를 구현해야 함
- Kafka Client의 Serializer/Deserializer에서도 활용하여 손쉽게 Custom객체의 Serializer/Deserializer 구현 가능
  - 참고로, Pure java의 API를 이용할 경우 Custom 객체의 멤버 변수 타입에 따른 바이트 배열 변환을 직접 수행해 줘야 함

## Kafka Client에서 Custom 객체의 직렬화/역직렬화 구현
- Jackson 활용 예제
  - 보통의 직렬화, 역직렬화시에 Json 객체로 만들어서 전송하는 경우가 많음 
  - Jackson 이란, Json 파일을 Java Object로, Java Object를 Json 파일로 변환 시키는 라이브러리
    - 아래 예제에서는 ObjectMapper를 사용하여 Java 객체를 JSON 바이트 배열로 직렬화하고, 역으로 JSON 데이터를 Java 객체로 역직렬화할 것임
	    - writeValueAsBytes(data): Java 객체를 JSON 바이트 배열로 변환
	    - readValue(data, CustomObject.class): JSON 바이트 배열을 지정한 클래스의 객체로 변환
	  - 장점
      - 사용법이 간단하고 직관적임
      - 다양한 JSON 구조를 지원하며, 커스터마이징 옵션이 풍부함
      - Kafka와 같이 메시지 직렬화가 중요한 환경에서 신뢰성 있게 작동함

- Custom 객체 생성
  - 직렬화 하려는 객체에 Getter/Setter 및 기본 생성자가 필요함

  ```java
    @Getter
    @Setter
    @ToString
    @NoArgsConstructor
    public class CustomObject {
        private int id;
        private String name;
    
        public CustomObject(int id, String name) {
            this.id = id;
            this.name = name;
        }
    }
  ```
  
- Serializer/Deserializer 구현

  ```java
    public class CustomObjectSerializer implements Serializer<CustomObject> {
        private final ObjectMapper objectMapper = new ObjectMapper();
    
        @Override
        public byte[] serialize(String topic, CustomObject data) {
            if (data == null) {
                return null;
            }
            try {
                // Jackson을 통해 CustomObject를 JSON 바이트 배열로 변환
                return objectMapper.writeValueAsBytes(data);
            } catch (Exception e) {
                throw new SerializationException("Error serializing CustomObject", e);
            }
        }
    }
  ```

- Kafka Client에 Custom 객체 Serialization/Deserialization 등록
  - Serialization/Deserialization 클래스를 Kafka Producer와 Consumer의 설정에 등록하여 커스텀 객체를 전송
  - Kafka Producer 설정 시, 커스텀 Serializer를 VALUE_SERIALIZER_CLASS_CONFIG에 등록
    - Consumer는 반대로, Deserializer를 등록해야함

  ```java
    public class KafkaCustomProducer {
        public static void main(String[] args) {
            Properties props = new Properties();
            props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
            props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
            props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, CustomObjectSerializer.class.getName());
    
            KafkaProducer<String, CustomObject> producer = new KafkaProducer<>(props);
            CustomObject customObj = new CustomObject(1, "Test Object");
    
            ProducerRecord<String, CustomObject> record = new ProducerRecord<>("custom_topic", "key1", customObj);
            producer.send(record, (metadata, exception) -> {
                if (exception == null) {
                    System.out.println("Message sent successfully to topic " + metadata.topic());
                } else {
                    exception.printStackTrace();
                }
            });
            producer.close();
        }
    }
  ```

- Jackson을 통해 데이터를 Json 타입으로 전송할 때의 문제점
  - 스키마 진화(Schema Evolution)에 대응하기 어려움
    - 스키마가 계속 변화되는 현상

### Avro 활용하기
![apache-avro](https://github.com/user-attachments/assets/27c7aba7-9547-4c13-ae25-497dd4480b21)

- Avro란, Apache에서 만든 데이터 직렬화(serialization) 프레임워크/시스템으로, 대규모 데이터 처리 환경에서 효율적이고 일관된 데이터 교환을 위해 설계되었음
- 특징
  - JSON 기반 스키마 정의
    - Avro의 스키마는 JSON 형식으로 정의되어 있어 사람이 쉽게 읽고 이해할 수 있음

  - 스키마 변경의 용이성
    - Avro는 스키마 진화를 지원하여 데이터 구조의 변경을 쉽게 처리할 수 있음
    - 즉, 새로운 필드를 추가하거나 기존 필드를 수정이 용이하며, 이는 데이터의 유연성과 상호 운용성을 높일 수 있음

  - 동적 코드 생성
    - Avro는 스키마를 사용하여 데이터를 직렬화하고 역직렬화하는 동적 코드 생성 기능을 제공함
    - 런타임 시에 스키마 정보를 사용하여 코드를 생성하므로 개발자가 직접 코드를 작성할 필요가 없음
    - 즉, JSON 으로 정의한 스키마를 자바 파일로 빌드하여 사용할 수 있어 개발이 용이하다는 장점을 갖음

  - 자체 압축 기능
    - Avro는 데이터를 압축하여 저장하는 기능을 제공함
    - 데이터의 전송 및 저장 과정에서 발생하는 네트워크 대역폭과 디스크 공간을 절약할 수 있음

  - Schema Registry와의 통합
    - Kafka와 Avro는 Confluent Schema Registry와 함께 사용되는 경우가 많음
    - Schema Registry란, 스키마를 중앙에서 관리하며, 생산자와 소비자가 같은 스키마를 공유함으로써 스키마 진화에 따른 문제를 효과적으로 해결

- kafka에서 동작 과정

  ![1_qYm6rGhRyINdeysqiylU5w](https://github.com/user-attachments/assets/c40dc1f7-ff59-4134-8e44-7142ed5ec1e5)

### 참고
- https://junior-datalist.tistory.com/384
- https://medium.com/nerd-for-tech/standardize-data-format-for-kafka-event-streams-using-apache-avro-and-schema-evolution-a2df6924b54c
