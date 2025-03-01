## 토픽의 세그먼트 관리

### 메시지 로그 세그먼트의 이해

- 세그먼트 (segment)
  - 카프카 로그 메시지 저장 단위
  - 카프카 파티션 디렉터리에 segment가 있음
- 세그먼트 저장
  - 파티션 당 N개의 segment로 구성됨
  - 각 segment는 데이터 용량, 일정 시간 설정에 따라 close 되고 새로 생성하면서 연속적으로 저장
- close된 세그먼트
  - 카프카 브로커가 write하지 않고 read-only 파일로 변경됨
- active한 세그먼트
  - 카프카 브로커는 N개의 세그먼트 중에 active 상태의 한 세그먼트에만 write/read
  - 하나의 파티션에는 하나의 active한 세그먼트가 있음

![image](https://github.com/user-attachments/assets/accb1277-c06b-4aa1-9a61-6034527e88ac)

### 세그먼트 저장 크기와 roll 관련 설정

- `log.segment.bytes`
  - 개별 segment의 최대 크기 (default: 1GB)
  - 최대 크기 넘길 시 close됨
- `log.roll.hours(ms)`
  - 개별 segment 최대 유지 시간 (default: 7day)
  - 최대 시간 넘길 시 close됨
  - 최대 크기만큼 차지 않아도 시간 넘기면 close

### 파티션 디렉터리 내 segment와 index 파일 구성

```
kafka-topics --bootstrap-server localhost:9092 --create -topic example-topic -- partitions


ll
total 16
-rw-r--r--  1 hyunseo  staff    10M Mar  1 16:41 00000000000000000000.index
-rw-r--r--  1 hyunseo  staff     0B Mar  1 16:41 00000000000000000000.log
-rw-r--r--  1 hyunseo  staff    10M Mar  1 16:41 00000000000000000000.timeindex
-rw-r--r--  1 hyunseo  staff     8B Mar  1 16:41 leader-epoch-checkpoint
-rw-r--r--  1 hyunseo  staff    43B Mar  1 16:41 partition.metadata
```

### 메시지 로그 segment와 index, TimeIndex

- index 파일
  - offset 별로 byte position 정보를 갖음
  - offset 데이터를 파일에서 읽기 위해서 시작 File Pointer에서 얼마만큼의 byte 위치에 있는지 알아야함.
- byte position
  - `log.index.interval.bytes` 설정 값만큼 segment bytes가 만들어질 때마다 offset을 byte position를 기록
- TimeIndex 파일
  - 생성 시간에 해당하는 offset 정보를 갖음 (millis)

![image](https://github.com/user-attachments/assets/34d262cd-f61d-4ce6-83b3-2fbb0218111d)

### segment 파일 rolling

- segment 파일은 rolling 되어 `active` -> `closed` 단계로 상태 변경됨
- `active` segment는 파티션 당 하나 존재

![image](https://github.com/user-attachments/assets/7df9ac4f-822a-40df-b234-22e0b854bb82)

### segment 파일 생명 주기

- `active` -> `closed` -> `deleted` 또는 `compacted`

### Log Cleanup Policy

카프카 브로커의 메시지 관리 정책

- `log.cleanup.policy=delete`
  - `log.retention.hours`, `log.retention.bytes` 설정 값에 따라 삭제
- `log.cleanup.policy=compact`
  - segment의 key 값 레벨로 가장 최신의 메시지만 유지
- `log.cleanup.policy=[delete,compact]`
  - compact와 delete 함께 적용
  
### Log Cleaner

- `log.cleaner.enabled=true`
  - Log Compaction 작업 수행 설정
- 클린 영역
  - Log Compaction 적용된 영역
- 더티 영역
  - Log Compaction 적용되지 않은 영역

![image](https://github.com/user-attachments/assets/30010133-d10f-44fd-906c-1c254d9024e0)

- Compact 대상
  - `active` segment 제외
- Compaction
  - 파티션 레벨에서 수행 결정
  - 개별 segment들을 새로운 segment로 재생성

![image](https://github.com/user-attachments/assets/a8a5add5-3903-4a5f-aa28-22343b86c200)

### Log Compaction

`log.cleanup.policy=compact` 설정 시 segment의 key 값에 따라 최신 메시지로만 segment 재구성

- 메시지 순서 유지
- 메시지 offset은 변하지 않음
- consumer는 메시지의 가장 최신값을 읽음

![image](https://github.com/user-attachments/assets/c2e7aa29-56a2-4ebe-83ba-4d3d170f5880)

### Log Compaction 수행

- `log.cleaner.min.cleanable.ratio`
  - Compaction 수행 조건이 되는 파티션 내 dirty 데이터 비율 (default: 0.5)
- `log.cleaner.min.compaction.lag.ms`
  - 메시지 생성 후 최소 ms가 지난 후 Compaction 대상 (default: 0)
- `log.cleaner.max.compaction.lag.ms`
  - dirty 데이터 비율 이하여도 ms 시간이 지나면 Compaction 대상 
