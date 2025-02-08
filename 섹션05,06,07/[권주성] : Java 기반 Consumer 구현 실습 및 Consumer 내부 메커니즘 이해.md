# 섹션 5-7 Java 기반 Consumer 구현 실습 및 Consumer 내부 메커니즘 이해

## Kafka Consumer 란?
- Kafka Consumer는 Kafka Broker로부터 특정 Topic(혹은 Partition)의 메시지를 읽어오는 역할을 담당하는 애플리케이션
  - 일반적으로 Kafka Consumer API를 사용하여 구현됨

    ![1_X1CBOxCjuuM5NDbvzcG4Gg](https://github.com/user-attachments/assets/4d7b20ce-f7e7-48e1-86a6-9d6f30716051)

### 주요 기능
- **`메시지 구독, 수신 및 처리`**
  - Kafka Broker에 특정 Topic을 구독(subscribe) 요청하고, 해당 Topic(혹은 Partition)에 저장된 메시지를 지속적으로 가져옴(polling)
    - 가져온 데이터를 기반으로 로그 분석, 이벤트 처리, 데이터 파이프라인 등 다양한 애플리케이션 로직을 처리(consume)
    - 메시지를 수신하지 않고 가져온다는 점이 특징

      <img width="500" alt="Kafka-Consumer-Figure-1" src="https://github.com/user-attachments/assets/92126e9c-750e-4a6f-94f3-e5f625ae8462" />

- **`Consumer Group 구성`**
  - 여러 Consumer가 하나의 Consumer Group을 이루어 병렬 처리가 가능해짐
    - 모든 Consumer들은 고유한 그룹아이디 group.id를 가지는 Consumer Group에 소속되어야 함
  - 동일 Group 내의 Consumer들은 각 Partition을 나누어 할당 받고 해당하는 Partition만 처리함
    - 동일 Consumer Group 내에서는 하나의 Partition은 하나의 Consumer에게만 할당될 수 있으며 이를 통해, 중복 처리를 방지함
  - 여러 Consumer Group이 같은 Topic을 구독할 수도 있으며, 이 경우 각 Group은 독립적으로 메시지를 소비함
 
    <img width="500" alt="1_Ehfc0FJvJUCxQ1F2v5D_yA" src="https://github.com/user-attachments/assets/443ca7a7-2931-46bd-9b8d-a309195b1d09" />

  - 각 Consumer들은 Heart Beat을 통해 자신의 상태를 Broker에 전송함. 만약 특정 Consumer가 종료되는 경우, Group Coordinators는 해당 Consumer가 처리하던 Partition을 다른 Consumer에서 할당하기 위한 재할당 과정을 수행(Consumer Rebalancing) 

    <img width="500" alt="CleanShot 2025-02-08 at 21 25 57@2x" src="https://github.com/user-attachments/assets/47aee114-252a-408b-a88d-3be52bf4d26e" />

- **`오프셋(Offset) 관리`**
  - Kafka는 Consumer가 메시지를 읽은 위치(offset)를 관리하여, 이를 기반으로 다시 시작할 위치를 지정할 수 있음
    - 이를 통해, 장애 발생 시 재시작 후에도 메시지를 이어서 소비할 수 있음
    - offset은 `__consumer_offsets` 이라고 하는 특수한 Kafka Topic을 통해 각 Consumer Group 마다 Partition 별로 관리되어짐
   
      <img width="500" alt="1" src="https://github.com/user-attachments/assets/4c414bb6-a575-48e8-a756-76bfbda5e29b" />

  - 자동 커밋(auto-commit) 혹은 수동 커밋(manual commit)을 통해 오프셋을 관리할 수 있음

## Kafka Consumer Client API
### Kafka Consumer Client Overview

  <img width="700" alt="commiting_offsets" src="https://github.com/user-attachments/assets/cc5a2f11-acb5-4c1e-a8f6-5545c45e77ad" />

### Kafka Consumer Client 주요 구성 요소 
  <img width="350" alt="CleanShot 2025-02-08 at 21 36 03@2x" src="https://github.com/user-attachments/assets/2a51d826-c4fb-42a9-8af5-645067d87e04" />

- `Fetcher`
  - Broker로부터 데이터를 가져오는(fetch) 역할을 담당
	- poll() 호출 시 fetch 요청을 전송하여 메시지를 가져옴
	- Broker의 특정 Partition으로부터 데이터를 가져오는 로직을 최적화
	- 배치 크기(fetch.min.bytes, fetch.max.bytes)를 조정하여 성능 최적화 가능

- `Network Client`
  - Broker와의 네트워크 통신하는 역할을 담당
  - Fetcher, Heartbeat Thread 등과 상호작용하여 비동기 I/O 방식으로 요청을 브로커에 전달하고 응답을 처리

- `Subscription State`
  - Consumer의 현재 상태를 관리하는 역할
  - Consumer가 구독한 토픽과 Partition 및 현재 처리 중인 Offset, Rebalancing 상태 등의 정보를 관리

- `Consumer Coordinator`
  - Consumer Group의 Partition 할당 조정 및 Rebalancing을 담당하는 역할

- `Heartbeat Thread`
  - Consumer 가 Consumer Group 내에서 정상적으로 활동하고 있음을 나타내는 하트비트(Heartbeat) 신호를 주기적으로 전송하는 역할

### Java Consumer Client API 처리 로직
- Consumer는 subscribe()를 호출하여 읽어 들이려는 토픽을 등록함
- Consumer는 poll( ) 메소드를 이용하여 주기적으로 브로커의 토픽 파티션에서 메시지를 가져옴
- 메시지를 성공적으로 가져 왔으면 commit을 통해서 __consumer_offse에 다음에 읽을 offset 위치를 기재함

  ![CleanShot 2025-02-08 at 22 41 23@2x](https://github.com/user-attachments/assets/140549da-360d-47ba-a3ec-658ae772f479)

- Consumer의 백그라운드에서는 별도의 스레드로 Heart Beat Thread가 Consumer의 정상적인 활동을 Group Coordinator에 보고하는 역할을 수행하고 있음

  ![CleanShot 2025-02-08 at 22 45 41@2x](https://github.com/user-attachments/assets/125dcd53-9857-4609-915f-c643f0386aec)

- 참고로, Consumer는 최초의 poll( ) 수행시 메시지를 가져오지 않고 여러가지 사전 과정을 수행함
  - Heart Beat Thread 생성, Broker에 MetaData 전달하여 Consumer Group에 Join, 등
  
### fetch 과정
![CleanShot 2025-02-08 at 23 00 21@2x](https://github.com/user-attachments/assets/26b35bcb-eea4-4fef-9572-cc5df7caf8fc)

- Linked Queue에 데이터가 있을 경우
  - Fetcher는 데이터를 가져오고 반환하며 poll() 수행 완료
  - ConsumerNetworkClient는 비동기로 계속 브로커의 메시지를 가져와서 Linked Queue에 저장

- Linked Queue에 데이터가 없을 경우
  - 1000ms 까지 Broker에 메시지 요청후 poll 수행() 완료
  - Fetcher는 Linked Queue 데이터를 가져오되, Linked Queue에 데이터가 없을 경우 ConsumerNetworkClient 에서 데이터를 브로커로 부터 가져올 것을 요청

### Consumer Fetcher 관련 주요 설정 파라미터 이해
- `fetch.min.bytes`
  - Fetcher가 record들을 읽어들이는 최소 bytes. 브로커는 지정된 fetch.min.bytes 이상의 새로운 메시지가 쌓일때 까지 전송을 하지 않음. 기본은 1
- `fetch.max.wait.ms`
  - 브로커에 fetch.min.bytes 이상의 메시지가 쌓일 때까지 최대 대기 시간. 기본은 500ms
- `fetch.max.bytes`
  - Fetcher가 한번에 가져올 수 있는 최대 데이터 bytes. 기본은 50MB
- `max.partition.fetch.bytes`
  - Fetcher가 파티션별 한번에 최대로 가져올 수 있는 bytes
- `max.poll.records`
  - Fetcher가 한번에 가져올 수 있는 레코드 수. 기본은 500
  
  ![0_rJcn6G9vLLTEoFox](https://github.com/user-attachments/assets/46bd6090-03ba-45df-aa43-46ad20f4d77e)

### Group Coordinator
- Consumer들의 Join Group 정보, Partition 매핑 정보 관리하는 역할

  ![CleanShot 2025-02-08 at 22 49 23@2x](https://github.com/user-attachments/assets/0704fee4-e4ed-439c-84f4-6f4266fa7634)

- Consumer 들의 HeartBeat 관리
  - Broker Group Coordinator는 주어진 시간동안 Heart Beat을 받지 못하면 Consumer들의 Rebalance를 수행 명령

### Consumer Rebalancing
- Consumer Group내에 새로운 Consumer가 추가되거나 기존 Consumer가 종료 될 때, 또는 Topic에 새로운 Partition이 추가될 때 Partition 할당을 새롭게 조정하기 위해 수행되는 과정
  - session.timeout.ms 이내에 Heartbeat이 응답이 없거나, max.poll.interval.ms 이내에 poll( ) 메소드가 호출되지 않을 경우에도 수행됨
- Broker의 Group Coordinator는 Consumer Group내의 Consumer들에게 Partition을 재 할당하는 Rebalancing을 수행하도록 지시

  ![CleanShot 2025-02-08 at 21 55 00@2x](https://github.com/user-attachments/assets/166e47e2-8c35-46fc-ae31-be64216b0d9c)

### Rebalancing 전략
- 각 버전에 따라 기본 모드가 다를 수 있음 

- `Eager 모드`
  - Rebalance 수행 시 기존 Consumer들의 모든 Partition 할당을 취소하고 잠시 메시지를 읽지 않음
  - 이후 새롭게 Consumer에 Partition을 다시 할당 받고 나서 다시 메시지를 읽음
  - 모든 Consumer가 잠시 메시지를 읽지 않는 시간으로 인해 Lag가 상대적으로 크게 발생할 가능성 있음
  
    ![CleanShot 2025-02-08 at 22 52 59@2x](https://github.com/user-attachments/assets/0e53e04a-4100-41b2-aa51-0193bc59c269)

- `Cooperative 모드`
  - Rebalance 수행 시 기존 Consumer들의 모든 Partition 할당을 취소하지 않고 대상이 되는 Consumer들에 대해서 Partition에 따라 점진적으로(Incremental) Consumer를 할당하면서 Rebalance를 수행하는 방식
  - 전체 Consumer가 메시지 읽기를 중지하지 않으며 개별 Consumer가 협력적으로(Cooperative) 영향을 받는 Partition만 Rebalance로 재 분배함
  - 많은 Consumer를 가지는 Consumer Group내에서 Rebalance 시간이 오래 걸릴 시 활용도 높음
  
    ![CleanShot 2025-02-08 at 22 53 21@2x](https://github.com/user-attachments/assets/426da44e-82fa-446d-9753-d7034a6c3431)

### Consumer 파티션 할당 전략
- `목표`
  - Consumer의 부하를 Partition 별로 균등하게 할당
  - 데이터 처리 및 리밸런싱의 효율성 극대화

- `Range`
  - 서로 다른 2개 이상의 토픽을 Consumer들이 Subscription 할 시 Topic별 동일한 Partition을 특정 Consumer에게 할당하는 전략
  - 여러 토픽들에서 동일한 키값으로 되어 있는 Partition은 특정 Consumer에 할당하여 해당 Consumer가 여러 토픽의 동일 키값으로 데이터 처리를 용이하게 할 수 있도록 지원

    ![range](https://github.com/user-attachments/assets/6c3bfb5c-d7bf-441b-b6f4-18ddd57b2aff)

- `Round Robin`
  - Partition 별로 Consumer들이 균등하게 부하를 분배할 수 있도록 여러 Topic들의 Partition들을 Consumer들에게 순차적인 Round robin 방식으로 할당

    ![roundrobin](https://github.com/user-attachments/assets/18b95e97-4b66-450e-b2ed-8f14d72ec502)

- `Sticky`
  - 최초에 할당된 Partition과 Consumer 매핑을 Rebalance 수행되어도 가급적 그대로 유지 할 수 있도록 지원하는 전략
  - 하지만 Eager Protocol Mode 기반이므로 Rebalance 시 모든 Consumer의 Partition 매핑이 해제 된 후에 다시 매핑되는 형태임

    ![stickey](https://github.com/user-attachments/assets/728cfadb-30a8-4cac-a4de-87b5265c5a11)

- `Cooperative Sticky`
  - 최초에 할당된 Partition과 Consumer 매핑을 Rebalance수행되어도 가급적 그대로 유지 할 수 있도록 지원함
  - 동시에 Cooperative Protocol Mode 기반으로 Rebalance 시 모든 Consumer의 Partition 매핑이 해제되지 않고 Rebalance 연관된 Partition과 Consumer만 재 매핑됨

    ![CleanShot 2025-02-08 at 23 10 40@2x](https://github.com/user-attachments/assets/0ffc64b7-eb8a-4ac0-904f-c1065e88fcb7)

### Consumer Static Group Membership
- Consumer Group내의 Consumer들에게 고정된 id를 부여
- Consumer 별로 Consumer Group 최초 조인 시 할당된 파티션을 그대로 유지하고 Consumer가 shutdown 되어도 session.timeout.ms내에 재 기동되면 rebalance가 수행되지 않고, 기존 파티션이 재 할당됨

  ![CleanShot 2025-02-08 at 22 51 32@2x](https://github.com/user-attachments/assets/2c422fa0-4387-490c-aa19-d38d636aeea7)


### Offset Commit의 이해
- Kafka는 **`__consumer_offsets`** 에는 Consumer Group이 특정 Topic의 Partition별로 읽기를 완수(commit)한 위치(offset) 정보를 가지고 있음 
  - 특정 Partition을 어느 Consumer가 commit 했는지 정보를 가지지 않음

    ![CleanShot 2025-02-08 at 22 35 10@2x](https://github.com/user-attachments/assets/115a5409-ed7b-47e8-a89f-b2d063248a96)

- Offset Commit을 잘못하는 경우 중복 읽기 혹은 읽기 누락의 상황이 발생할 수 있음
  - 이미 메시지가 처리가 되었으나 Commit이 안되어 있는 상태에서 종료된 경우 → Commit이 완료된 메시지 부터 읽기 때문에 중복 읽기 발생

    ![CleanShot 2025-02-08 at 22 07 14@2x](https://github.com/user-attachments/assets/dd3471fa-42ea-46a9-992a-d4febab5d1b6)

  - 아직 메시지가 처리 되지 않았으나 먼저 Commit이 된 상태에서 종료된 경우 → Commit이 완료된 메시지 부터 읽기 때문에 읽기 누락 발생
  
    ![CleanShot 2025-02-08 at 22 07 25@2x](https://github.com/user-attachments/assets/60e872f0-c441-478d-91af-4961e83cf1da)

- Auto Offset 과 Manual Offset 두가지 방식을 제공함

### Auto Commit
- 사용자가 명시적으로 코드로 commit을 기술하지 않아도 Consumer가 자동으로 지정된 기간마다 commit을 수행
- auto.enable.commit=true 인 경우에 동작
- 읽어온 메시지를 브로커에 바로 commit 적용하지 않고, `auto.commit.interval.ms` 에 정해진 주기(기본 값 5초)마다 consumer가 자동으로 commit을 수행함
  - 정해진 주기 이후 다음 poll() 동작시 commit 요청도 같이 수행 
- consumer가 읽어온 메시지보다 브로커의 commit이 오래 되었으므로 consumer의 장애/재기동 및 rebalancing 후 브로커에서 이미 읽어온 메시지를 다시 읽어와서 중복 처리 될 수 있음

  ![CleanShot 2025-02-08 at 22 19 10@2x](https://github.com/user-attachments/assets/f95c3966-fcef-4ca3-b600-bae3f2ff0011)

### Manual Commit
- 사용자가 명시적으로 commit을 하는 방식
- Sync, Async 두가지 방식이 있음 

- **`Sync(동기 방식)`**
  - `commitSync()` 메소드를 사용
  - poll( )을 통해서 읽어온 메시지 배치에 해당 메시지들의 마지막 offset을 commit함
  - broker 에 commit 적용 요청이 성공적으로 완료될 때까지 블로킹이 됨
  - commit 적용 완료 후에 다시 메시지를 읽어오기 시작
  - 만약, 브로커에 commit 적용이 실패할 경우 다시 commit 적용 요청 
  - 비동기 방식 대비 더 느린 수행 시간

    ![CleanShot 2025-02-08 at 22 09 30@2x](https://github.com/user-attachments/assets/daebb6df-5bd6-47bf-aa83-b98c4be80347)

- **`ASync(비동기 방식)`**
  - `commitAsync()` 메소드를 사용
  - poll( )을 통해서 읽어온 메시지 배치에 해당 메시지들의 마지막 offset을 commit함
  - broker 에 commit 적용이 성공적으로 되었음을 기다리지 않고(블로킹하지 않음) 계속 메시지를 읽어옴
  - broker에 commit 적용이 실패해도 다시 commit 시도 하지 않음
    - 장애 또는 Rebalance 시 한번 읽은 메시지를 다시 중복해서 가져올 수도 있음
    - 콜백 함수를 실패시 다시 재시도를 할 수 있으나, 비동기 적으로 동작하기 때문에 실패한 commit 요청 이후에 다음 commit 요청은 성공했을 수도 있기 때문에 주의가 필요함
  - 동기 방식 대비 더 빠른 수행 시간

    ![CleanShot 2025-02-08 at 22 09 38@2x](https://github.com/user-attachments/assets/e497e5e9-2f33-4213-b287-762f574e2913)


## 참고
- https://blog.developer.adobe.com/exploring-kafka-consumers-internals-b0b9becaa106
- https://d2.naver.com/helloworld/0974525

