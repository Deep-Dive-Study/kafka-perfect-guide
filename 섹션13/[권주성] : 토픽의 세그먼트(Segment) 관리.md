# 섹션13 토픽의 세그먼트(Segment) 관리

## 카프카 로그의 파티션과 세그먼트
- **카프카의 로그 메시지는 실제로는 `Segment(세그먼트)` 라는 단위의 파일 형태로 저장이 됨**
  - 즉, 파티션은 논리적 형태인 파일 디렉토리로 존재하며, 해당 파티션 디렉토리 안에 메시지 저장 segment를 file로서 가지고 있음

- 파티션은 여러 개의 segment 들로 구성되며 개별 segment는 데이터 용량이 차거나 혹은 일정 시간이 경과하면 close 되고 새로운 segment를 생성하여 데이터를 연속적으로 저장함
  - segment가 close 상태가 되면 더 이상 브로커가 write 하지 않고 read-only 로서 존재하게 됨
  - 브로커는 여러 개의 segment 중에 단 하나의 active segment에만 write와 read를 수행할 수 있음
  - 하나의 파티션은 단 하나의 active segment를 가짐

  ![Adv_Kafka_Topic_Internals_2](https://github.com/user-attachments/assets/3b7b026a-10da-4593-bc45-2c3e96d4db76)

### Segment 파일의 Rolling
- **Segment 파일은 Rolling 되어서 `Active` → `Closed` 단계로 상태가 변경됨**
  - 아래에 나와 있는 속성값에 따라, 데이터 용량이 차거나 혹은 일정 시간이 경과하게 되면 Rolling 수행 
  - 따라서, 하나의 파티션에는 `단 하나의 Active Segment만 존재` 

- segment 파일명은 해당 segment 별로 시작 offset 을 기반으로 작성됨

  ![CleanShot 2025-02-26 at 00 35 23@2x](https://github.com/user-attachments/assets/d1661767-234b-4933-a14e-8c3f420b3fb2)

### Segment의 저장 크기와 Roll 관련 설정

<table>
    <tr>
        <th>설정 항목</th>
        <th>설명</th>
    </tr>
    <tr>
        <td rowspan="3"><strong>log.segment.bytes</strong></td>
        <td><b>개별 segment의 최대 크기 (기본값: 1GB)<b></td>
    </tr>
    <tr>
        <td>지정된 크기를 초과하면 해당 segment는 더 이상 active segment가 아니고 close됨 (write 불가, read 가능)</td>
    </tr>
    <tr>
        <td>Topic 설정에서는 <code>segment.bytes</code>를 사용하며, 기본적으로 <code>log.segment.bytes</code> 값을 따름</td>
    </tr>
    <tr>
        <td rowspan="4"><strong>log.roll.hours (ms)</strong></td>
        <td><b>개별 segment가 유지되는 최대 시간 (기본값: 7일)<b></td>
    </tr>
    <tr>
        <td>지정된 시간을 초과하면 해당 segment는 더 이상 active segment가 아니고 close됨 (write 불가, read 가능)</td>
    </tr>
    <tr>
        <td><code>log.segment.bytes</code> 크기만큼 차지하지 않아도 <code>log.roll.ms</code> 시간이 지나면 해당 segment를 close함</td>
    </tr>
    <tr>
        <td>Topic 설정에서는 <code>segment.ms</code>를 사용하며, 기본적으로 <code>log.roll.hours</code> 값을 따름</td>
    </tr>
</table>

## 파티션 디렉토리내의 Segment와 Index 파일 구성
- Topic을 생성시 파티션 디렉토리내는 3가지 종류의 파일이 생성됨
  - `메시지 내용을 가지는 segment`
  - `offset의 위치 byte 정보를 가지는 index 파일`
  - `record 생성 시간에 따른 위치 byte 정보를 가지는 timeindex 파일`

  ![1_Mo4rS67NlMTUSpRf_JEUYA](https://github.com/user-attachments/assets/61449d0b-fc0f-404e-ba05-d78355b606eb)

- **`일반 Index 파일은 offset 별로 byte position 정보를 가지고 있음`**
  - 메시지 Segment는 File 기반이므로 특정 offset의 데이터를 파일에서 읽기 위해서는 시작 File Pointer에서 얼마만큼의 byte에 위치해 있는지 알아야함
  - index 파일은 모든 offset에 대한 byte position 정보를 가지고 있지 않으며 `log.index.interval.bytes` 에 설정된 값만큼의 segment bytes가 만들어질 때마다 해당 offset에 대한 byte position 정보를 기록함

- **`Time Index 파일은 메시지의 생성 Unix 시간을 밀리 세컨드 단위로 가지고 있고 해당 생성 시간에 해당하는 offset 정보를 가짐`**

  ![1_yMgNoiK3cXdOhQOZAoiveQ](https://github.com/user-attachments/assets/222f4eae-5980-42db-b451-64a334effdc3)

- **`참고`**
  - snapshot 파일은 중복 레코드를 피하기 위해 사용된 시퀀스 ID와 관련된 생산자 상태의 스냅샷이 들어 있음
    - 새로운 리더가 선출된 후 선호되는 리더가 돌아와서 다시 리더가 되기 위해 그러한 상태가 필요할 때 사용됨

## Segment 파일의 생명 주기
- segment 파일은 `Active` → `Closed` → `Deleted` 또는 `Compacted` 단계로 관리됨
- segment 파일은 `Log Cleanup Policy`에 따라 지정된 특정 시간이나 파일 크기에 따라 Delete 되거나 Compact 형태로 저장됨

### Log Cleanup Policy (로그 관리 정책)
- 카프카 브로커는 오래된 메시지를 관리하기 위한 정책을 `log.cleanup.policy` 로 설정
  - Topic 레벨으로 설정하려면 `cleanup.policy` 속성을 설정하면됨

- `log.cleanup.policy = delete` 로 설정하면 segment를 `log.retention.hours` 나 `log.retention.bytes` 설정 값에 따라 삭제함
- `log.cleanup.policy = compact` 로 설정하면 segment를 key 값 레벨로 가장 최신의 메시지만 유지하도록 segment 재구성함
- `log.cleanup.policy = [delete, compact]` 로 설정하면 compact와 delete를 함께 적용할 수 있음

#### Log.cleanup.policy=delete시 삭제를 위한 설정 파라미터

<table>
    <tr>
        <th>설정 항목</th>
        <th>설명</th>
    </tr>
    <tr>
        <td rowspan="4"><strong>log.retention.hours (ms)</strong></td>
        <td><b>개별 Segment가 삭제되기 전 유지하는 시간 (기본값: 1주일, 168시간)<b></td>
    </tr>
    <tr>
        <td>크게 설정하면 오래된 segment를 그대로 유지하므로 디스크 공간이 더 필요함</td>
    </tr>
    <tr>
        <td>작게 설정하면 오래된 segment를 조회할 수 없음</td>
    </tr>
    <tr>
        <td>Topic 설정에서는 <code>retention.ms</code>를 사용하며, 기본적으로 <code>log.retention.hours</code> 값을 따름</td>
    </tr>
    <tr>
        <td rowspan="3"><strong>log.retention.bytes</strong></td>
        <td><b>Segment 삭제 조건이 되는 파티션 단위의 전체 파일 크기 설정 (기본값: <code>-1</code>, 무한대)<b></td>
    </tr>
    <tr>
        <td>적정한 디스크 공간 사용량을 제약하기 위해 보통 설정함</td>
    </tr>
    <tr>
        <td>Topic 설정에서는 <code>retention.bytes</code>를 사용하며, 기본적으로 <code>log.retention.bytes</code> 값을 따름</td>
    </tr>
    <tr>
        <td rowspan="1"><strong>log.retention.check.interval.ms</strong></td>
        <td><b>브로커가 백그라운드에서 Segment 삭제 대상을 찾는 주기 (ms 단위)<b></td>
    </tr>
</table>


## Log Compaction 이란?
- `log.cleanup.policy=compact` 로 설정 시 **`Segment의 내에 있는 로그 메시지의 Key 값에 따라 가장 최신의 메시지로만 compact하게 segment 재구성하게 됨`**
  - `Key값이 null인 메시지에는 적용할 수 없음`
 
    ![kafka-log-compaction-process (1)](https://github.com/user-attachments/assets/ae105133-d64e-4589-afde-cc2b3531e5e1)

- 백그라운드 스레드 방식으로 별도의 I/O 작업을 수행하므로 추가적인 I/O 부하가 소모됨
- Log 파일은 Compaction 수행을 위한 기준으로 보았을때 두가지로 나뉨
  - `클린(clean) 영역` : Log Compaction이 이미 적용됨
  - `더티(dirty) 영역` : Log Compaction이 아직 적용되지 않음
  
### Log Compaction 수행
- log compaction 작업 수행하기 위해서는 `log.cleaner.enabled=true` 설정이 되어야함
  - 해당 설정은 true 로 두어야함. 카프카는 내부에 `__consumer_offsets` 토픽을 관리하는데 log compaction을 적용하고 있음
- active segment는 compact 대상에서 제외됨
- compaction은 파티션 레벨에서 수행 결정이 되며, 개별 세그먼트들을 새로운 세그먼트들로 재 생성함
  - compaction 작업으로 인해 메시지가 사라질 수 있기 때문에 compaction을 적용해도 괜찮은(최신 값만 유효한 경우) 경우에 적용해야함
 
- log compaction 작업은 Dirty 비율이 `log.cleaner.min.cleanable.ratio` 이상이고 메시지가 생성된 지 l`og.cleaner.min.compaction.lag.ms`이 지난 Dirty 메시지에 수행됨
  - 혹은, 메시지가 생성된지 `log.cleaner.max.compaction.lag.ms` 이 지난 Dirty 메시지에 수행 

  ![Kafka_Internals_112](https://github.com/user-attachments/assets/a02d4926-9c6c-4211-8d4c-885c7294e49b)

  ![Kafka_Internals_113](https://github.com/user-attachments/assets/ac0156e2-1cb8-4127-818c-c27058f781c6)

  ![Kafka_Internals_114](https://github.com/user-attachments/assets/6a2a45fd-ba30-4282-b025-4ad4c994e815)

  ![Kafka_Internals_115](https://github.com/user-attachments/assets/0037a761-35f2-4ebc-9430-071b5098e21f)

#### Log Compaction 수행 이후
- 메시지의 순서(Ordering)은 여전히 유지됨
- 메시지의 offset도 변하지 않음
- consumer는 (active segment를 제외하고) 메시지의 가장 최신값을 읽음
  - 단, Compact 기반으로 로그가 재생될 시 키값은 있지만 밸류가 Null 값을 가지는 메시지는 일정 시간 이후에 삭제 되기 때문에 읽지 못할 수 있음
    - 밸류가 Null 값을 가지는 메시지는 내부적으로 tombston으로 표기하면 삭제 처리함

      ![CleanShot 2025-02-26 at 01 34 25@2x](https://github.com/user-attachments/assets/b763fa3a-a0e3-4540-b0ae-3e4760477ae7)

#### Log Compaction 설정

<table>
    <tr>
        <th>설정 항목</th>
        <th>설명</th>
    </tr>
    <tr>
        <td rowspan="3"><strong>log.cleaner.min.cleanable.ratio</strong></td>
        <td>Log cleaner가 Compaction을 수행하기 위한 파티션 내의 dirty 데이터 비율 (dirty/total)</td>
    </tr>
    <tr>
        <td>기본값은 0.5이며, 값이 작을수록 Log cleaner가 더 빈번히 Compaction을 수행</td>
    </tr>
    <tr>
        <td>Topic 설정에서는 <code>min.cleanable.dirty.ratio</code>를 사용</td>
    </tr>
    <tr>
        <td rowspan="3"><strong>log.cleaner.min.compaction.lag.ms</strong></td>
        <td>메시지가 생성된 후 최소한 해당 시간이 지나야 Compaction 대상이 될 수 있음</td>
    </tr>
    <tr>
        <td>기본값은 0이며, 값이 클수록 최근 데이터보다는 오래된 데이터를 기준으로 수행</td>
    </tr>
    <tr>
        <td>Topic 설정에서는 <code>min.compaction.lag.ms</code>를 사용</td>
    </tr>
    <tr>
        <td rowspan="3"><strong>log.cleaner.max.compaction.lag.ms</strong></td>
        <td>Dirty ratio 이하라도 메시지 생성 후 해당 시간이 지나면 Compaction 대상이 될 수 있음</td>
    </tr>
    <tr>
        <td>기본값은 무한대에 가까운 큰 값</td>
    </tr>
    <tr>
        <td>Topic 설정에서는 <code>max.compaction.lag.ms</code>를 사용</td>
    </tr>
</table>

### LSM Tree
- 참고로 해당 Log Compaction 작업과 매우 유사한 형태로 데이터를 관리하는 자료 구조가 있음
- 찾아보니 카프카에서 LSM Tree를 사용하는 것은 아님(데이터를 정렬할 필요가 없기 때문)
- 그러나, 유사한 시스템으로 데이터를 관리하기 때문에 한번 찾아보면 도움이 될 것 같음
  - https://www.youtube.com/watch?v=i_vmkaR1x-I
  - https://blog.devgenius.io/log-structured-merge-tree-a8733ce152b2 

  ![0_JeaDeiI9ncQMrqGK](https://github.com/user-attachments/assets/ac3b3af7-801b-4595-87ee-935aa06ef8a8)

## 참고
- https://medium.com/geekculture/kafka-internals-part-2-7dad1977f7d1
- https://strimzi.io/blog/2021/12/17/kafka-segment-retention/
- https://developer.confluent.io/courses/architecture/compaction/
