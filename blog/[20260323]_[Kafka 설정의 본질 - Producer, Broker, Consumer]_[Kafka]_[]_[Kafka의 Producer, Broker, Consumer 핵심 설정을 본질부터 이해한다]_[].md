# Kafka 설정의 본질 — Producer, Broker, Consumer

## 들어가며

Kafka 설정을 검색하면 수십 개의 프로퍼티 목록이 나온다. `acks`, `retries`, `batch.size`, `linger.ms`... 하나하나 외우려 하면 끝이 없다.

그런데 이 설정들을 뜯어보면, 결국 **세 가지 질문**으로 수렴한다.

1. **메시지를 잃어버려도 되는가?** (신뢰성)
2. **얼마나 빨라야 하는가?** (성능)
3. **실패하면 어떻게 할 것인가?** (장애 대응)

설정값을 외우는 게 아니라, **"이 설정이 어떤 질문에 대한 답인가"**를 이해하면 수십 개의 프로퍼티가 몇 가지 원칙으로 정리된다.

이 글에서는 Producer, Broker, Consumer 순서로 핵심 설정을 정리하되, 각 설정이 **왜 존재하는지**, 그리고 **어떤 트레이드오프를 강제하는지**를 중심으로 다룬다.

---

## Producer — "보냈다"의 기준을 정하는 쪽

Producer의 모든 설정은 결국 하나의 질문으로 귀결된다.

> **"메시지를 보냈다"는 게 정확히 무슨 뜻인가?**

### acks — "확인"의 범위를 정한다

```properties
# 리더만 확인하면 성공
acks=1

# 리더 + 모든 ISR(In-Sync Replica)이 확인해야 성공
acks=all

# 확인 안 기다림 (보내고 끝)
acks=0
```

이 설정 하나가 Kafka Producer의 성격을 결정한다.

| 설정 | 의미 | 메시지 유실 가능성 | 속도 |
|------|------|-------------------|------|
| `acks=0` | 보내고 끝. 응답 안 기다림 | 유실 가능 | 가장 빠름 |
| `acks=1` | 리더가 받으면 성공 | 리더 장애 시 유실 가능 | 보통 |
| `acks=all` | 리더 + ISR 전부 복제 완료 | 유실 거의 없음 | 가장 느림 |

`acks=all`이 무조건 좋은 게 아니다. 로그 수집처럼 "좀 빠지더라도 빠르게 보내야 하는" 경우에는 `acks=0`이 맞다. **"이 메시지가 얼마나 중요한가?"가 기준이다.**

```
[acks=1일 때 리더 장애 시나리오]

Producer → Leader (저장 완료, acks=1 응답) → Follower (아직 복제 안 됨)
                     ↓ (리더 장애 발생)
              Follower가 새 리더로 승격
                     ↓
              해당 메시지는 사라짐 — Producer는 성공이라 믿고 있음
```

결제 이벤트라면? `acks=all`이어야 한다. 클릭 로그라면? `acks=1`이면 충분하다. **설정값이 아니라 비즈니스 요구사항이 답을 정한다.**

### retries + delivery.timeout.ms — 실패했을 때의 태도

```properties
# 재시도 횟수 (기본값: 2147483647, 사실상 무한)
retries=2147483647

# 전체 전송 타임아웃 (기본값: 120000ms = 2분)
delivery.timeout.ms=120000

# 재시도 간격 (기본값: 100ms)
retry.backoff.ms=100
```

Kafka Producer의 기본 `retries`가 사실상 무한이라는 걸 처음 알면 놀란다. 이건 **"재시도 횟수가 아니라 시간으로 제한하겠다"**는 설계 철학이다.

```
[delivery.timeout.ms의 의미]

delivery.timeout.ms = linger.ms + request.timeout.ms + (retries × retry.backoff.ms) + α

┌─────────────────── delivery.timeout.ms (전체 시간 제한) ───────────────────┐
│                                                                            │
│  [배치 대기] → [전송 시도1] → [실패] → [대기] → [전송 시도2] → [실패] → ... │
│  linger.ms    request.timeout   retry.backoff    request.timeout           │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘

→ delivery.timeout.ms 초과 시 → TimeoutException 발생
```

`retries`를 3으로 줄이면 3번 실패 후 바로 포기한다. `delivery.timeout.ms`를 5초로 줄이면 5초 안에 안 되면 포기한다. **횟수로 제한할 것인가, 시간으로 제한할 것인가**의 차이다.

실무에서는 `delivery.timeout.ms`로 제한하는 편이 낫다. "3번 실패"보다 "5초 안에 안 되면 포기"가 비즈니스 관점에서 이해하기 쉽기 때문이다.

### enable.idempotence — 재시도해도 중복되지 않게

```properties
# 멱등성 활성화 (Kafka 3.0+ 기본값: true)
enable.idempotence=true

# 멱등성을 위해 필요한 조건
acks=all
retries > 0
max.in.flight.requests.per.connection <= 5
```

재시도를 하면 중복이 생길 수 있다. Producer가 메시지를 보냈는데 응답을 못 받아서 다시 보내면, Broker에 같은 메시지가 두 번 저장된다.

```
[멱등성 비활성화 시]

Producer → Broker (메시지 저장 완료)
        ← (응답 유실 — 네트워크 문제)
Producer → Broker (같은 메시지 다시 전송)
        → Broker에 같은 메시지 2건 저장됨

[멱등성 활성화 시]

Producer → Broker (메시지 저장, PID=1, Seq=0)
        ← (응답 유실)
Producer → Broker (같은 메시지 재전송, PID=1, Seq=0)
        → Broker: "PID=1, Seq=0은 이미 있다. 무시." → 중복 저장 안 됨
```

Producer는 고유한 Producer ID(PID)와 Sequence Number를 메시지에 붙인다. Broker는 이미 받은 (PID, Seq) 조합이면 저장하지 않는다. **재시도를 안전하게 만드는 장치다.**

### batch.size + linger.ms — 모아서 보내기의 두 가지 기준

```properties
# 배치 크기 (기본값: 16384 = 16KB)
batch.size=16384

# 배치 대기 시간 (기본값: 0ms)
linger.ms=0

# 전체 버퍼 메모리 (기본값: 33554432 = 32MB)
buffer.memory=33554432
```

Producer는 메시지를 하나씩 보내지 않는다. **모아서 한 번에 보낸다.** 그 기준이 두 가지다.

```
[배치 전송 흐름]

send() 호출 → RecordAccumulator (버퍼)에 쌓임
                    ↓
              어떤 조건이 먼저 충족되면 전송:
              1. batch.size 도달 → 바로 전송
              2. linger.ms 경과 → 크기 상관없이 전송
                    ↓
              Sender 스레드가 Broker로 전송
```

| 설정 | 역할 | 트레이드오프 |
|------|------|-------------|
| `batch.size` | 배치 크기 상한 | 크면 처리량↑, 메모리↑ |
| `linger.ms` | 배치 대기 시간 | 길면 처리량↑, 지연↑ |

`linger.ms=0`(기본값)이면 메시지가 들어오자마자 즉시 전송한다. 배치 효과를 보려면 `linger.ms`를 5~50ms 정도 주는 것이 일반적이다.

```java
// Spring Kafka Producer 설정 예제
@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

        // 신뢰성: 모든 ISR이 확인해야 성공
        config.put(ProducerConfig.ACKS_CONFIG, "all");

        // 멱등성: 재시도해도 중복 없음
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);

        // 배치: 32KB 모이거나 20ms 지나면 전송
        config.put(ProducerConfig.BATCH_SIZE_CONFIG, 32768);
        config.put(ProducerConfig.LINGER_MS_CONFIG, 20);

        // 전송 타임아웃: 30초 안에 안 되면 포기
        config.put(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, 30000);

        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

```java
// 메시지 전송 예제
@Service
@RequiredArgsConstructor
public class OrderEventProducer {

    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;

    public void sendOrderCreatedEvent(OrderCreatedEvent event) {
        String payload = objectMapper.writeValueAsString(event);

        // key를 orderId로 → 같은 주문은 항상 같은 파티션으로
        kafkaTemplate.send("order-events", event.getOrderId(), payload)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("주문 이벤트 전송 실패: orderId={}", event.getOrderId(), ex);
                    // 실패 처리: DB에 기록하고 재시도 스케줄링
                } else {
                    log.info("주문 이벤트 전송 성공: topic={}, partition={}, offset={}",
                        result.getRecordMetadata().topic(),
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                }
            });
    }
}
```

### compression.type — 네트워크 비용을 CPU로 치환한다

```properties
# 압축 방식 (기본값: none)
compression.type=lz4
```

| 방식 | 압축률 | 속도 | 용도 |
|------|--------|------|------|
| `none` | 없음 | 가장 빠름 | 메시지가 작을 때 |
| `gzip` | 높음 | 느림 | 네트워크 대역폭 절약이 중요할 때 |
| `snappy` | 보통 | 빠름 | 범용 |
| `lz4` | 보통 | 가장 빠름 | 처리량 중심 |
| `zstd` | 가장 높음 | 보통 | 압축률이 중요할 때 |

압축은 네트워크 대역폭과 CPU 사이의 트레이드오프다. 메시지가 JSON처럼 텍스트 기반이면 압축 효과가 크다. 실무에서는 **`lz4`** 또는 **`snappy`**가 균형이 좋다.

### Producer 설정 요약

```
질문: "메시지를 잃어버려도 되는가?"
  → acks=all, enable.idempotence=true (잃으면 안 된다)
  → acks=0 (좀 잃어도 된다)

질문: "얼마나 빨라야 하는가?"
  → batch.size↑, linger.ms↑, compression.type=lz4 (처리량 우선)
  → linger.ms=0, batch.size↓ (지연 최소화)

질문: "실패하면 어떻게 할 것인가?"
  → delivery.timeout.ms로 시간 제한
  → 콜백에서 실패 처리 로직
```

---

## Broker — 메시지를 "얼마나 안전하게, 얼마나 오래" 보관할 것인가

Broker는 메시지의 **저장소**다. 설정의 핵심은 두 가지다.

1. 복제를 어디까지 보장할 것인가
2. 데이터를 얼마나 오래 보관할 것인가

### min.insync.replicas — acks=all의 진짜 의미를 결정한다

```properties
# 최소 동기화 복제본 수 (기본값: 1)
min.insync.replicas=2
```

`acks=all`은 "모든 ISR이 확인"이라고 했다. 그런데 ISR이 리더 하나뿐이면? `acks=all`이어도 리더만 확인하는 셈이다. **`min.insync.replicas`가 `acks=all`의 실제 강도를 결정한다.**

```
[replication.factor=3, min.insync.replicas=2일 때]

Producer → Leader (저장) → Follower1 (복제 완료) → Follower2 (복제 중...)
                                    ↓
                        ISR = {Leader, Follower1, Follower2}
                        min.insync.replicas = 2
                        → Leader + Follower1이 확인 → acks=all 충족 → 성공 응답

[Follower2가 장애로 ISR에서 빠진 경우]

ISR = {Leader, Follower1}
min.insync.replicas = 2
→ Leader + Follower1 확인 → 여전히 충족 → 정상 동작

[Follower1도 장애로 ISR에서 빠진 경우]

ISR = {Leader}
min.insync.replicas = 2
→ Leader만 있음, ISR 크기 < min.insync.replicas
→ NotEnoughReplicasException → Producer 쓰기 거부
```

이게 핵심이다. **데이터를 잃느니 차라리 쓰기를 거부한다.** 이것이 `min.insync.replicas`의 태도다.

실무에서 가장 일반적인 조합은 이렇다.

```properties
# Broker 설정 (server.properties)
default.replication.factor=3
min.insync.replicas=2

# Producer 설정
acks=all
```

복제본 3개, 최소 2개 동기화, 전부 확인. 이러면 **1대가 죽어도 데이터 유실 없이 서비스가 계속된다.**

### log.retention — 데이터의 수명을 결정한다

```properties
# 시간 기반 보관 (기본값: 168시간 = 7일)
log.retention.hours=168

# 크기 기반 보관 (기본값: -1 = 무제한)
log.retention.bytes=-1

# 세그먼트 파일 크기 (기본값: 1073741824 = 1GB)
log.segment.bytes=1073741824

# 삭제 정책 (기본값: delete)
log.cleanup.policy=delete
```

Kafka는 메시지를 Consumer가 읽어도 삭제하지 않는다. **보관 기간이 지나야 삭제한다.** 이것이 RabbitMQ 같은 전통 메시지 큐와의 근본적인 차이다.

```
[log.retention.hours=168 (7일)]

Day 1: 메시지 A 저장 ─────────────────────────── Day 8: 메시지 A 삭제
Day 2: 메시지 B 저장 ─────────────────────────── Day 9: 메시지 B 삭제
  ...
Day 7: 메시지 G 저장 ─────────────────────────── Day 14: 메시지 G 삭제

→ Consumer가 3일 전 메시지를 다시 읽고 싶다? 가능하다.
→ Consumer가 10일 전 메시지를 읽고 싶다? 이미 삭제됐다.
```

| 설정 | 용도 |
|------|------|
| `log.retention.hours=168` | 일반적인 이벤트 스트림 (7일) |
| `log.retention.hours=720` | 감사 로그, 규정 준수 (30일) |
| `log.cleanup.policy=compact` | 최신 상태만 유지 (KTable, 설정 토픽) |

### log.cleanup.policy=compact — 삭제가 아니라 압축

```properties
log.cleanup.policy=compact
```

`compact`는 같은 key를 가진 메시지 중 **마지막 값만 남긴다.**

```
[Compaction 전]
Offset 0: key=user-1, value={name: "김철수", age: 25}
Offset 1: key=user-2, value={name: "이영희", age: 30}
Offset 2: key=user-1, value={name: "김철수", age: 26}  ← user-1 업데이트
Offset 3: key=user-1, value={name: "김철수", age: 27}  ← user-1 또 업데이트

[Compaction 후]
Offset 1: key=user-2, value={name: "이영희", age: 30}  ← 최신 값
Offset 3: key=user-1, value={name: "김철수", age: 27}  ← 최신 값

→ user-1의 과거 이력(25, 26)은 사라지고, 마지막 상태(27)만 남는다.
```

이 정책은 **"히스토리가 아니라 현재 상태가 중요한"** 토픽에 쓴다. 사용자 프로필, 설정값, Kafka Streams의 KTable 같은 것들.

### num.partitions — 병렬성의 상한을 결정한다

```properties
# 토픽 기본 파티션 수 (기본값: 1)
num.partitions=6
```

파티션 수는 **Consumer의 최대 병렬 처리 수**를 결정한다. Consumer Group에서 하나의 파티션은 하나의 Consumer만 읽을 수 있기 때문이다.

```
[파티션 3개, Consumer 3개]
Partition 0 → Consumer A  ✓ (1:1 매핑, 최대 효율)
Partition 1 → Consumer B  ✓
Partition 2 → Consumer C  ✓

[파티션 3개, Consumer 5개]
Partition 0 → Consumer A  ✓
Partition 1 → Consumer B  ✓
Partition 2 → Consumer C  ✓
             Consumer D  ✗ (놀고 있음)
             Consumer E  ✗ (놀고 있음)

→ Consumer를 아무리 늘려도, 파티션 수 이상으로는 병렬화되지 않는다.
```

**파티션은 늘리기는 쉽지만 줄이기는 어렵다.** 줄이려면 토픽을 새로 만들어야 한다. 그래서 처음부터 넉넉하게 잡되, 불필요하게 크게 잡지 않는 것이 중요하다.

### unclean.leader.election.enable — 데이터를 잃을 수 있어도 서비스를 살릴 것인가

```properties
# ISR 외 복제본의 리더 선출 허용 (기본값: false)
unclean.leader.election.enable=false
```

ISR에 속한 모든 Broker가 죽으면, 리더를 뽑을 수 없다. 이때 두 가지 선택이 있다.

```
[모든 ISR이 죽은 상태]

선택 1: unclean.leader.election.enable=false (기본값)
  → 리더 없음 → 해당 파티션 읽기/쓰기 불가 → 가용성 포기, 데이터 보존

선택 2: unclean.leader.election.enable=true
  → ISR 밖의 복제본을 리더로 선출 → 서비스 재개 → 하지만 동기화 안 된 메시지는 유실
```

**데이터 유실 vs 서비스 중단.** 결제 시스템이라면 `false`가 맞다. 로그 수집이라면 `true`가 나을 수 있다. 이 역시 비즈니스 요구사항이 답을 정한다.

### Broker 핵심 설정 정리

```properties
# === server.properties 예제 ===

# 복제 및 안정성
default.replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false

# 파티션
num.partitions=6

# 보관 정책
log.retention.hours=168
log.retention.bytes=-1
log.segment.bytes=1073741824
log.cleanup.policy=delete

# 네트워크 / IO
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# 메시지 크기
message.max.bytes=1048576
```

---

## Consumer — "읽었다"의 기준을 정하는 쪽

Consumer의 핵심 질문은 이것이다.

> **"메시지를 읽었다"는 걸 언제 확정하는가?**

이 질문에 대한 답이 offset commit이고, 이 commit을 어떻게 하느냐에 따라 **중복 처리**와 **유실**의 양이 달라진다.

### enable.auto.commit — 자동으로 "읽었다"고 말할 것인가

```properties
# 자동 커밋 (기본값: true)
enable.auto.commit=true

# 자동 커밋 간격 (기본값: 5000ms)
auto.commit.interval.ms=5000
```

자동 커밋은 5초마다 "여기까지 읽었습니다"라고 Broker에 알린다. 편하지만 위험하다.

```
[자동 커밋의 문제 — 메시지 유실]

1. Consumer가 offset 10까지 poll()
2. 자동 커밋 실행 → "offset 10까지 읽었음" 기록
3. 메시지 7~10 처리 중 Consumer 장애
4. 재시작 → offset 10부터 읽기 시작
5. 메시지 7~10은 처리 안 됐는데, 이미 커밋됨 → 유실

[자동 커밋의 문제 — 중복 처리]

1. Consumer가 offset 10까지 poll()
2. 메시지 7~10 처리 완료
3. 아직 자동 커밋 안 됨 (5초 안 지남)
4. Consumer 장애
5. 재시작 → 마지막 커밋된 offset 6부터 읽기 시작
6. 메시지 7~10 다시 처리 → 중복
```

**자동 커밋은 "처리 완료"와 "커밋"이 동기화되지 않는다.** 이것이 근본적인 문제다.

### 수동 커밋 — 처리가 끝난 후에 직접 확정한다

```properties
enable.auto.commit=false
```

```java
// Spring Kafka 수동 커밋 설정
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "order-service");
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);

        // 수동 커밋
        config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        // 한 번에 가져올 최대 레코드 수
        config.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 100);

        // poll 간격 제한 (이 시간 안에 다음 poll을 해야 함)
        config.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000);

        return new DefaultKafkaConsumerFactory<>(config);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());

        // MANUAL: 처리 완료 후 직접 ack 호출
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);

        return factory;
    }
}
```

```java
// 수동 커밋 Consumer 예제
@Service
public class OrderEventConsumer {

    @KafkaListener(topics = "order-events", groupId = "order-service")
    public void consume(ConsumerRecord<String, String> record, Acknowledgment ack) {
        try {
            OrderCreatedEvent event = objectMapper.readValue(record.value(), OrderCreatedEvent.class);

            // 비즈니스 로직 처리
            orderService.processOrder(event);

            // 처리가 완료된 후에만 커밋
            ack.acknowledge();

            log.info("주문 이벤트 처리 완료: orderId={}, offset={}",
                event.getOrderId(), record.offset());

        } catch (Exception e) {
            log.error("주문 이벤트 처리 실패: offset={}", record.offset(), e);
            // ack를 호출하지 않으면 → 다음 poll에서 다시 받음
            // 하지만 무한 실패 루프에 빠질 수 있으므로 DLT(Dead Letter Topic)로 보내는 것이 좋음
        }
    }
}
```

```
[수동 커밋 흐름]

1. poll() → 메시지 수신
2. 비즈니스 로직 처리
3. 처리 성공 → ack.acknowledge() → offset 커밋
4. 처리 실패 → ack 안 함 → 다음 poll에서 다시 수신

→ "처리 완료"와 "커밋"이 동기화된다.
```

### max.poll.records + max.poll.interval.ms — Consumer의 심장박동

```properties
# 한 번의 poll()에 가져올 최대 레코드 (기본값: 500)
max.poll.records=500

# poll() 간 최대 간격 (기본값: 300000ms = 5분)
max.poll.interval.ms=300000

# 하트비트 간격 (기본값: 3000ms)
heartbeat.interval.ms=3000

# 세션 타임아웃 (기본값: 45000ms)
session.timeout.ms=45000
```

이 네 가지 설정이 Consumer의 "살아있음"을 결정한다.

```
[Consumer 리밸런싱이 발생하는 경우]

Broker: "Consumer A, 살아있어?"

경우 1: session.timeout.ms (45초) 동안 하트비트 없음
  → Broker: "Consumer A 죽은 것 같다. 리밸런싱 시작."

경우 2: max.poll.interval.ms (5분) 동안 poll() 호출 없음
  → Broker: "Consumer A가 처리를 멈춘 것 같다. 리밸런싱 시작."
```

`max.poll.records`를 너무 크게 잡으면 처리 시간이 길어져서 `max.poll.interval.ms`를 초과할 수 있다. 그러면 Consumer가 살아있는데도 리밸런싱이 발생한다.

```
[잘못된 설정 예시]

max.poll.records=10000          # 한 번에 1만 건
max.poll.interval.ms=300000     # 5분 제한

→ 1만 건 처리에 6분 걸림 → 5분 초과 → 리밸런싱 → 다시 1만 건 수신 → 무한 루프
```

**`max.poll.records`와 `max.poll.interval.ms`는 반드시 함께 조정해야 한다.** 1건당 처리 시간 × `max.poll.records` < `max.poll.interval.ms`가 되어야 한다.

### auto.offset.reset — Consumer가 처음이거나, offset이 사라졌을 때

```properties
# offset 없을 때 시작 위치 (기본값: latest)
auto.offset.reset=latest
```

| 설정 | 동작 | 용도 |
|------|------|------|
| `latest` | 지금 이후의 메시지만 읽음 | 실시간 처리, 과거 무시 |
| `earliest` | 처음부터 다 읽음 | 데이터 파이프라인, 전수 처리 |
| `none` | offset 없으면 예외 발생 | 엄격한 관리가 필요할 때 |

새로운 Consumer Group이 토픽을 구독할 때, 처음부터 읽을지 지금부터 읽을지를 정한다. **과거 데이터가 중요하면 `earliest`, 실시간만 중요하면 `latest`.**

### Dead Letter Topic (DLT) — 실패한 메시지의 무덤이 아니라 격리소

처리에 실패한 메시지를 계속 재시도하면 Consumer가 멈춘다. 이때 DLT로 보내서 나중에 처리한다.

```java
// Spring Kafka DLT 설정 예제
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);

        // 3번 재시도 후 DLT로 전송
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate(),
                (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition())),
            new FixedBackOff(1000L, 3L)  // 1초 간격, 3번 재시도
        ));

        return factory;
    }
}
```

```java
// DLT Consumer — 실패한 메시지를 별도로 처리
@Service
public class DeadLetterConsumer {

    @KafkaListener(topics = "order-events.DLT", groupId = "dlt-handler")
    public void handleDeadLetter(ConsumerRecord<String, String> record, Acknowledgment ack) {
        log.warn("DLT 메시지 수신: topic={}, offset={}, value={}",
            record.topic(), record.offset(), record.value());

        // 알림 발송, DB 기록 등
        alertService.notifyDeadLetter(record);

        ack.acknowledge();
    }
}
```

```
[DLT 흐름]

메시지 수신 → 처리 실패 → 재시도 1 (1초 후) → 실패
                                → 재시도 2 (1초 후) → 실패
                                → 재시도 3 (1초 후) → 실패
                                → DLT(order-events.DLT)로 전송
                                → 원본 토픽의 offset 커밋 (다음 메시지 진행)

→ 실패한 메시지가 정상 메시지의 처리를 막지 않는다.
```

### Consumer 설정 요약

```
질문: "메시지를 잃어버려도 되는가?"
  → enable.auto.commit=false + 수동 ack (잃으면 안 된다)
  → enable.auto.commit=true (약간의 유실 감수)

질문: "중복 처리를 어떻게 할 것인가?"
  → 멱등한 처리 로직 설계 (DB upsert, unique constraint 등)
  → 중복은 Kafka의 at-least-once에서 피할 수 없다. 애플리케이션에서 막는다.

질문: "실패하면 어떻게 할 것인가?"
  → DLT로 격리 → 별도 처리
  → max.poll.records와 max.poll.interval.ms 조정으로 리밸런싱 방지
```

---

## 전체 설정 조합 — 시나리오별 가이드

### 시나리오 1: 결제 이벤트 (메시지 유실 절대 불가)

```properties
# Producer
acks=all
enable.idempotence=true
delivery.timeout.ms=30000
retries=2147483647

# Broker (Topic)
replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false

# Consumer
enable.auto.commit=false
auto.offset.reset=earliest
max.poll.records=50
```

### 시나리오 2: 클릭 로그 수집 (처리량 우선, 약간의 유실 허용)

```properties
# Producer
acks=1
batch.size=65536
linger.ms=50
compression.type=lz4

# Broker (Topic)
replication.factor=2
min.insync.replicas=1
log.retention.hours=72

# Consumer
enable.auto.commit=true
auto.commit.interval.ms=5000
max.poll.records=1000
```

### 시나리오 3: 사용자 프로필 동기화 (최신 상태만 중요)

```properties
# Producer
acks=all
enable.idempotence=true

# Broker (Topic)
replication.factor=3
min.insync.replicas=2
log.cleanup.policy=compact

# Consumer
enable.auto.commit=false
auto.offset.reset=earliest
```

---

## 마무리 — 설정은 질문에 대한 답이다

Kafka 설정이 수십 개지만, 결국 세 가지 질문에 대한 답이다.

> **메시지를 잃어버려도 되는가?**
> - Producer: `acks`, `enable.idempotence`
> - Broker: `replication.factor`, `min.insync.replicas`
> - Consumer: `enable.auto.commit`, 수동 ack

> **얼마나 빨라야 하는가?**
> - Producer: `batch.size`, `linger.ms`, `compression.type`
> - Broker: `num.partitions`
> - Consumer: `max.poll.records`, `fetch.min.bytes`

> **실패하면 어떻게 할 것인가?**
> - Producer: `delivery.timeout.ms`, `retries`, 전송 콜백
> - Broker: `unclean.leader.election.enable`
> - Consumer: DLT, `max.poll.interval.ms`

설정값을 외우려 하지 말자. **"이 시스템에서 메시지가 얼마나 중요한가"를 먼저 정하면, 설정은 자연스럽게 따라온다.**

기술 문서에는 기본값과 설명이 있다. 하지만 정작 중요한 건 **"왜 이 값이어야 하는가"**다. 그 "왜"는 Kafka 문서가 아니라 우리 서비스의 요구사항에 있다.
