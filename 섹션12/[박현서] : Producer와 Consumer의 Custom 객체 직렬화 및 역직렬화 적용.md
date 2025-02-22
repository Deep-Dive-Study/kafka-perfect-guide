## Producer와 Consumer의 Custom 객체 직렬화/역직렬화

### Serializer

```java
public class ExampleProducer {
    public static void main(String[] args) {
        String topicName = "example-topic";

        Properties props = new Properties();
        props.setProperty("bootstrap.servers", "localhost:9092");
        props.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, OrderSerializer.class.getName()); // serializer 등록
        
        // ...
    }
}
```

```java
package com.practice.kafka.producer;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import com.practice.kafka.model.OrderModel;
import org.apache.kafka.common.serialization.Serializer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class OrderSerializer implements Serializer<OrderModel> {
    private static final Logger logger = LoggerFactory.getLogger(OrderSerializer.class);

    private static final ObjectMapper objectMapper = new ObjectMapper().registerModule(new JavaTimeModule());

    @Override
    public byte[] serialize(final String topic, final OrderModel order) {
        byte[] serializedOrder = null;

        try {
            serializedOrder = objectMapper.writeValueAsBytes(order);
        } catch (JsonProcessingException e) {
            logger.error(e);
        }

        return serializedOrder;
    }
}
```

### Deserializer

```java
public class ExampleConsumer {
    public static void main(java.lang.String[] args) {
        java.lang.String topicName = "example-topic";

        Properties props = new Properties();
        props.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.setProperty(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.setProperty(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, OrderDeserializer.class.getName()); // deserializer 등록
        props.setProperty(ConsumerConfig.GROUP_ID_CONFIG, "file-group");
        
        // ...
    }
}
```

```java
package com.practice.kafka.consumer;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import com.practice.kafka.model.OrderModel;
import org.apache.kafka.common.serialization.Deserializer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;

public class OrderDeserializer implements Deserializer<OrderModel> {
    private static final Logger logger = LoggerFactory.getLogger(OrderDeserializer.class);
    private static final ObjectMapper objectMapper = new ObjectMapper().registerModule(new JavaTimeModule());

    @Override
    public OrderModel deserialize(final String topic, final byte[] data) {
        OrderModel orderModel = null;

        try {
            orderModel = objectMapper.readValue(data, OrderModel.class);
        } catch (IOException e) {
            logger.error(e);
        }

        return orderModel;
    }
}
```

### String(Json), Avro

|  | **문자열 직렬화 (JSON)** | **바이너리 직렬화 (Avro)** |
|---|---|---|
| **파일 크기** | 큼 (텍스트 포함) | 작음 (메타데이터 최소화) |
| **네트워크 트래픽** | 많음 | 적음 |
| **직렬화 속도** | 느림 | 빠름 |
| **역직렬화 속도** | 느림 (JSON 파싱 필요) | 빠름 (바이트 변환) |
| **가독성** | 높음 (사람이 읽을 수 있음) | 없음 (바이너리) |
| **스키마 변경** | 유연 (필드 추가 가능) | 스키마 등록 필요 |
| **언어 독립성** | 높음 | 높음 (하지만 JSON보다 복잡) |
| **사용 사례** | 설정 파일, API 응답 | Kafka, 고성능 데이터 처리 |


- **문자열 직렬화 (JSON) 추천**
  - 사람이 데이터를 직접 읽고 수정해야 하는 경우 (설정 파일, 로그, API 응답).
  - 데이터 크기가 크지 않고, 네트워크 성능이 큰 문제가 되지 않는 경우.
  - 언어 간 호환성이 가장 중요할 때 (REST API, 웹 애플리케이션).
- **바이너리 직렬화 (Avro) 추천**
  - **Kafka, gRPC, 데이터 스트리밍, 대량의 데이터 처리**가 필요한 경우.
  - 네트워크 트래픽을 줄이고 빠른 역직렬화가 필요한 경우.
  - 스키마를 엄격하게 관리하고, 데이터 무결성을 보장해야 하는 경우.
