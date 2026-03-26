# Kafka 튜닝의 본질 — 조건이 옵션을 결정한다

## 들어가며

[이전 글](https://zzangoobrother.github.io/BLOG/?post=%5B20260323%5D_%5BKafka%20%EC%84%A4%EC%A0%95%EC%9D%98%20%EB%B3%B8%EC%A7%88%20-%20Producer%2C%20Broker%2C%20Consumer%5D_%5BKafka%5D_%5B%5D_%5BKafka%EC%9D%98%20Producer%2C%20Broker%2C%20Consumer%20%ED%95%B5%EC%8B%AC%20%EC%84%A4%EC%A0%95%EC%9D%84%20%EB%B3%B8%EC%A7%88%EB%B6%80%ED%84%B0%20%EC%9D%B4%ED%95%B4%ED%95%9C%EB%8B%A4%5D_%5B%5D.md)에서는 Kafka의 핵심 설정을 **"왜 존재하는가"**라는 관점에서 정리했다. `acks`, `retries`, `batch.size` 같은 설정이 어떤 질문에 대한 답인지를 다뤘다.

이번 글은 그다음 단계다. 설정의 의미를 안다고 해서 튜닝을 할 수 있는 건 아니다. **구체적인 시스템 조건이 주어졌을 때, 어떤 값을 어떻게 정해야 하는가**가 진짜 문제다.

Kafka 튜닝을 검색하면 "batch.size는 이렇게, linger.ms는 저렇게"라는 레시피가 넘쳐난다. 그대로 따라 하면 운이 좋으면 되고, 운이 나쁘면 오히려 장애가 난다. 옵션값은 "내 시스템의 조건"에 대한 답이지, 보편적인 정답이 아니기 때문이다.

이 글에서는 **숫자를 먼저 읽고, 그 숫자가 가리키는 병목을 찾고, 병목을 풀어주는 옵션을 도출하는 사고 과정**을 다룬다. "이 값이 좋다"가 아니라 **"이 조건에서는 이 값이어야 하는 이유"**에 집중한다.

---

## 조건 정리

| 항목 | 값 |
|------|-----|
| 메시지 크기 | 평균 1MB, 최대 5MB |
| 브로커 메모리 | 2GB |
| 초당 발행 메시지 수 | 평균 10,000개 |
| 컨슈머 1개당 처리 속도 | 1,000개/초 (CPU 100%) |
| 컨슈머 메시지 처리 시간 | 0.01초/개 |
| 순서 보장 | 불필요 |

### 목표

1. 브로커 메모리 다운 방지
2. 네트워크 통신 최소화
3. CPU 과부하 방지

이 세 가지 목표는 서로 **긴장 관계**에 있다.

```
압축 → 네트워크 ↓ 하지만 CPU ↑
배치 → 네트워크 ↓ 하지만 메모리 ↑
병렬화 → CPU ↓ 하지만 네트워크 ↑ (컨슈머마다 fetch 발생)
```

어느 하나를 극단적으로 밀면 다른 하나가 터진다. **세 목표의 교집합을 찾는 것**이 튜닝의 본질이다.

---

## 숫자부터 읽자 — 세 가지 병목 도출

옵션을 건드리기 전에, 조건에서 핵심 수치를 뽑아야 한다.

### 병목 1: 초당 데이터량

```
10,000 msgs/sec × 1MB = 10GB/sec
```

비압축 기준으로 **초당 10GB**가 브로커를 관통한다. 2GB 메모리 브로커에게 이 트래픽은 데이터가 메모리에 0.2초도 머물 수 없다는 뜻이다.

→ **압축이 선택이 아니라 필수다.**

### 병목 2: 컨슈머 CPU

```
1 컨슈머 = 1,000 msgs/sec → CPU 100%
목표 CPU 50% → 1,000 × 0.5 = 500 msgs/sec
필요 컨슈머 수 = 10,000 / 500 = 20개
필요 파티션 수 ≥ 20 → 20개
```

CPU 100%는 GC 한 번이면 장애다. 50%로 낮추면 메시지 크기 편차(1MB~5MB)와 스파이크를 흡수할 여유가 생긴다.

→ **한 번에 처리하는 양을 절반으로 줄이고, 대수를 늘려야 한다.**

### 병목 3: 브로커 페이지 캐시 체류 시간

```
브로커 메모리: 2GB
JVM 힙: 1GB → 나머지 OS 페이지 캐시: 1GB

초당 유입 데이터: 10GB (비압축)
→ 압축 적용 시 (lz4, 약 2배): 5GB/sec

페이지 캐시 체류 시간: 1GB / 5GB = 0.2초
```

압축을 해도 데이터가 페이지 캐시에 **0.2초** 머문다. 컨슈머가 이 안에 소비하지 못하면 디스크에서 읽어야 하고, 그 순간 지연이 폭증한다.

→ **컨슈머는 브로커의 페이지 캐시보다 빨리 소비해야 한다.**

![세 병목의 관계](img/kafka_bottleneck_flow.svg)

---

## Producer — 같은 데이터를 더 적은 요청으로 보내라

프로듀서 튜닝의 목표는 명확하다. **네트워크 요청 수를 줄이면서, 브로커에 걸리는 메모리 부담을 낮추는 것.**

### compression.type — 가장 큰 한 방

```properties
compression.type=lz4
```

압축 하나로 세 가지 문제가 동시에 완화된다.

```
[압축 전] 10,000 msgs/sec × 1MB = 10GB/sec
[압축 후] 10,000 msgs/sec × ~0.5MB = ~5GB/sec

→ 네트워크 대역폭 50% 절감
→ 브로커 디스크 I/O 50% 절감
→ 브로커 페이지 캐시 체류 시간 0.1초 → 0.2초 (2배 개선)
```

**왜 lz4인가?**

| 코덱 | 압축률 | 압축 속도 | 해제 속도 | 이 조건에서의 판단 |
|------|--------|----------|----------|------------------|
| gzip | 높음 | 느림 | 느림 | CPU 여유 없음 → 탈락 |
| snappy | 중간 | 빠름 | 빠름 | 후보 |
| **lz4** | **중간** | **매우 빠름** | **매우 빠름** | **CPU 병목 시 최적** |
| zstd | 높음 | 빠름 | 빠름 | 압축률 좋지만 해제 비용이 lz4보다 높음 |

컨슈머 CPU가 100%인 상황에서 **해제 속도가 가장 빠른 코덱**을 선택해야 한다. lz4의 해제 속도는 gzip 대비 10배 이상 빠르다.

> **핵심**: 압축은 프로듀서에서 하고, 브로커는 그대로 저장하고, 컨슈머에서 해제한다. **브로커 CPU는 압축에 거의 관여하지 않는다.** 비용은 프로듀서와 컨슈머가 나눠 가진다.

### batch.size + linger.ms — 요청 수를 줄여라

```properties
batch.size=5242880
linger.ms=5
```

**왜 5MB인가?**

기본값(16KB)은 1MB 메시지 하나도 담지 못한다. 5MB면 평균 크기 메시지 5개를 하나의 배치로 묶을 수 있다.

```
[배치 효과 계산]

파티션 수: 20
파티션당 메시지 유입: 10,000 / 20 = 500 msgs/sec
메시지 평균 크기: 1MB

배치 없이:
  네트워크 요청 = 10,000 requests/sec

batch.size=5MB (메시지 5개):
  파티션당 배치 전송 = 500 / 5 = 100 batches/sec
  전체 요청 = 100 × 20 = 2,000 requests/sec

→ 네트워크 요청 80% 감소
```

**linger.ms=5의 근거:**

```
파티션당 500 msgs/sec → 메시지 간격 = 2ms
batch.size 5MB (5개) 채우는 시간: 2ms × 5 = 10ms
```

linger.ms=5면 5ms 시점에 배치가 절반쯤 차 있어도 전송한다. 배치가 먼저 차면 linger.ms를 기다리지 않고 즉시 전송하므로, **5ms는 상한선이지 지연이 아니다.** 레이턴시보다 처리량을 우선하는 설정이다.

### max.request.size — 메시지 거부를 막아라

```properties
max.request.size=10485760
```

최대 메시지 크기가 5MB이고 batch.size가 5MB이므로, 단일 요청에 배치 + 헤더가 들어갈 수 있도록 10MB로 설정한다.

```
max.request.size < 최대 메시지 크기 → MessageSizeTooLargeException
max.request.size < batch.size → 배치 전송 불가

→ max.request.size ≥ batch.size + α 가 되어야 안전하다.
```

### buffer.memory — 프로듀서 내부 버퍼

```properties
buffer.memory=134217728
```

프로듀서는 전송 전에 메시지를 내부 버퍼(RecordAccumulator)에 쌓는다.

```
최소 필요 버퍼 = 파티션 수 × batch.size = 20 × 5MB = 100MB
일시적 브로커 응답 지연 고려 → 약간의 여유 → 128MB
```

버퍼가 부족하면 `send()` 호출이 블로킹되거나 `BufferExhaustedException`이 발생한다.

### acks — 순서 보장 불필요의 이점

```properties
acks=1
```

순서 보장이 필요 없으므로 리더만 확인하는 `acks=1`로 충분하다.

```
acks=0: 확인 안 함 → 유실 가능 → 너무 위험
acks=1: 리더 확인 → 리더 장애 시에만 유실 → 성능과 안정의 균형
acks=all: 모든 ISR 확인 → 가장 안전하지만 레이턴시 증가
```

`acks=all`이면 ISR 복제 완료까지 기다려야 하므로 프로듀서의 전송 지연이 늘어난다. 초당 10,000건을 보내는 상황에서 이 지연은 buffer.memory 소진으로 이어질 수 있다.

### Producer 설정 요약

```properties
# 압축: 네트워크 50% 절감, lz4로 해제 비용 최소화
compression.type=lz4

# 배치: 네트워크 요청 80% 감소 (10,000 → 2,000 req/sec)
batch.size=5242880
linger.ms=5

# 메시지 크기: 최대 5MB 메시지 + 배치 수용
max.request.size=10485760

# 버퍼: 20파티션 × 5MB + 여유분
buffer.memory=134217728

# 신뢰성: 리더 확인 (순서 불필요이므로 acks=1)
acks=1
```

---

## Kafka Broker — 통로를 막지 마라

2GB 메모리 브로커에 10GB/sec(압축 후 5GB/sec)가 쏟아진다. 브로커는 데이터를 **저장하는 곳이 아니라 통과시키는 곳**으로 생각해야 한다.

### 메모리 배분 — JVM 힙과 페이지 캐시

```
# JVM 옵션
KAFKA_HEAP_OPTS="-Xmx1g -Xms1g"
```

Kafka 브로커의 메모리 구조를 이해해야 한다.

![2GB 브로커 메모리 배분](img/kafka_2gb_broker_memory.svg)

**핵심 원리**: Kafka는 메시지를 JVM 힙이 아니라 **OS 페이지 캐시**를 통해 처리한다. 프로듀서가 보낸 데이터는 페이지 캐시에 기록되고, 컨슈머가 최신 데이터를 요청하면 페이지 캐시에서 직접 전달한다(zero-copy). **JVM 힙은 1GB면 메타데이터 관리에 충분하다.**

### message.max.bytes — 프로듀서와 맞춰라

```properties
message.max.bytes=5242880
replica.fetch.max.bytes=5242880
```

```
프로듀서 max.request.size > 브로커 message.max.bytes
→ 브로커가 메시지를 거부한다.

replica.fetch.max.bytes < message.max.bytes
→ 팔로워가 리더에서 메시지를 복제하지 못한다.

→ 세 값은 반드시 정합성을 맞춰야 한다.
  max.request.size ≥ message.max.bytes = replica.fetch.max.bytes
```

### queued.max.requests — 메모리 다운 방지의 핵심

```properties
queued.max.requests=100
```

**이 옵션이 브로커 메모리 다운을 막는 가장 중요한 설정이다.**

브로커는 프로듀서/컨슈머의 요청을 내부 큐에 쌓아두고 순서대로 처리한다. 이 큐의 크기가 `queued.max.requests`다.

```
[기본값 500일 때]

프로듀서 요청 = 압축된 배치 약 2.5MB/건 (5MB / 2)

큐에 쌓이는 최대 데이터:
  500 × 2.5MB = 1,250MB → JVM 힙 1GB 초과 → OOM 위험!

[100으로 줄이면]

  100 × 2.5MB = 250MB → JVM 힙의 25%

나머지 750MB:
  → 컨슈머 그룹 관리, 리밸런싱, GC에 충분한 여유.
```

> **본질**: `queued.max.requests`는 "브로커가 동시에 들고 있을 수 있는 요청의 총량"을 결정한다. 메모리가 작을수록 이 값을 낮춰야 한다.

### num.partitions — 컨슈머 병렬도의 상한

```properties
num.partitions=20
```

앞서 계산한 결과:

```
필요 컨슈머: 20개
파티션: 20개

[파티션-컨슈머 매핑]

파티션 0  → 컨슈머 0   (1:1 매핑)
파티션 1  → 컨슈머 1
...
파티션 19 → 컨슈머 19

→ 1:1 매핑이 가장 이상적인 상태다.
```

순서 보장이 불필요하므로 파티션을 늘려도 순서 문제가 없다. 실무에서는 리밸런싱 버퍼와 스케일 아웃 여유를 위해 컨슈머 수보다 **20~50%** 더 잡는 것이 일반적이다. **파티션은 늘리기 쉽지만 줄이기 어렵다.**

### I/O 스레드 — 처리 용량 확보

```properties
num.network.threads=8
num.io.threads=16
```

```
[스레드 역할]

네트워크 스레드 (num.network.threads):
  → 요청/응답 소켓 I/O 담당
  → 기본값 3 → 2,000 req/sec + 컨슈머 fetch 처리에는 부족
  → 8로 증가

I/O 스레드 (num.io.threads):
  → 디스크 읽기/쓰기, 요청 처리 담당
  → 기본값 8 → 5GB/sec 디스크 쓰기에는 부족
  → 16으로 증가

※ 실제 값은 브로커 CPU 코어 수에 따라 조정.
  네트워크 스레드: 코어 수 이하
  I/O 스레드: 코어 수의 2배 이하
```

### log.retention — 디스크를 지켜라

```properties
log.retention.hours=24
```

압축 후에도 초당 5GB가 디스크에 쌓인다.

```
시간당 디스크 사용량: 5GB × 3,600 = 18TB/hour

retention 7일(기본값): 18TB × 168h = 3,024TB → 비현실적
retention 24시간:      18TB × 24h  = 432TB   → 여전히 크지만 관리 가능
retention 6시간:       18TB × 6h   = 108TB   → 합리적
```

보관 기간은 디스크 용량과 비즈니스 요구사항에 따라 결정한다. 10GB/sec 트래픽에서 기본 7일 보관은 사실상 불가능하다.

### Broker 설정 요약

```properties
# JVM
KAFKA_HEAP_OPTS="-Xmx1g -Xms1g"

# 메시지 크기
message.max.bytes=5242880
replica.fetch.max.bytes=5242880

# 메모리 보호: 인메모리 요청 큐 제한
queued.max.requests=100

# 병렬성
num.partitions=20

# 처리 스레드
num.network.threads=8
num.io.threads=16

# 보관: 디스크 용량에 맞게 조정
log.retention.hours=24
```

---

## Consumer — 한 번에 덜 먹어라

컨슈머의 문제는 명확하다. **1,000개를 한 번에 처리하면 CPU가 100%까지 치솟는다.**

### max.poll.records — CPU 제어의 핵심

```properties
max.poll.records=500
```

**왜 500인가?**

```
현재: max.poll.records 조정 전 → 1,000 msgs/sec → CPU 100%
조정: max.poll.records=500     → ~500 msgs/sec  → CPU ≈ 50%

처리량을 절반으로 줄이면, CPU도 절반이 된다.
```

CPU 50%가 주는 것:

```
- 최대 메시지(5MB)가 연속으로 올 때 흡수 여유: 50%
- GC pause 발생 시 흡수 여유: 50%
- 피크 트래픽 시 버퍼: 50%
```

CPU 100%에서는 5MB 메시지가 몇 건만 연속으로 와도 처리 시간이 급증하여 타임아웃이 발생한다. **평균이 아니라 피크를 견딜 수 있는 여유**가 있어야 한다.

```
[max.poll.records와 CPU의 관계]

                  ┌─ 목표 지점
                  ▼
CPU ■■■■■■■■■■■■■■■■■■■■ 100%  (1,000개)
CPU ■■■■■■■■■■░░░░░░░░░░  50%  (500개)  ← 선택
```

### 컨슈머 수와 파티션의 관계

```
필요 처리량: 10,000 msgs/sec
컨슈머당 처리량: 500 msgs/sec (max.poll.records=500)
필요 컨슈머 수: 10,000 / 500 = 20개
파티션 수: 20개

[파티션-컨슈머 매핑]

파티션 0  → 컨슈머 0
파티션 1  → 컨슈머 1
...
파티션 19 → 컨슈머 19

→ 20파티션 : 20컨슈머 = 1:1 매핑 (최적 상태)
→ 컨슈머를 더 늘려도 파티션이 20개이므로 효과 없음
```

### fetch 설정 — 네트워크 통신 최소화

```properties
fetch.min.bytes=1048576
fetch.max.wait.ms=200
fetch.max.bytes=104857600
max.partition.fetch.bytes=10485760
```

네 가지 설정이 **컨슈머가 브로커에서 데이터를 가져오는 방식**을 제어한다.

```
[fetch 흐름]

컨슈머 → 브로커: "데이터 주세요"

브로커:
  데이터 < fetch.min.bytes (1MB)?
    ├─ YES → 기다림...
    │         └─ fetch.max.wait.ms (200ms) 초과?
    │              ├─ YES → 있는 만큼 응답
    │              └─ NO  → 계속 기다림
    └─ NO  → 즉시 응답

  응답 크기 제한:
    전체: fetch.max.bytes (100MB)
    파티션당: max.partition.fetch.bytes (10MB)
```

**fetch.min.bytes=1MB**: "최소 1MB가 쌓일 때까지 응답하지 마라." 잦은 빈 응답(empty fetch)을 없애서 네트워크 왕복을 줄인다.

**fetch.max.wait.ms=200ms**: 1MB가 안 쌓여도 200ms가 지나면 응답한다. 처리 지연의 상한선.

**fetch.max.bytes=100MB**: 한 번의 fetch 응답 최대 크기. lz4 압축 상태로 전달되므로 실제 네트워크 전송량은 ~50MB다.

**max.partition.fetch.bytes=10MB**: 파티션당 최대 fetch 크기. 특정 파티션에 데이터가 몰려도 한 번에 과도한 데이터를 가져오지 않도록 방어한다.

```
[fetch.min.bytes의 효과]

fetch.min.bytes=1 (기본값):
  → 데이터가 1바이트만 있어도 응답
  → 초당 수백 번의 fetch 요청 발생 가능
  → 네트워크 낭비

fetch.min.bytes=1MB:
  → 1MB가 쌓일 때까지 대기
  → fetch 빈도 대폭 감소
  → 네트워크 효율 ↑

※ 대신 최대 200ms의 추가 지연이 발생할 수 있다.
  실시간성이 극도로 중요하면 이 값을 낮춰야 한다.
```

### session.timeout.ms — 리밸런싱을 막아라

```properties
session.timeout.ms=30000
heartbeat.interval.ms=10000
max.poll.interval.ms=300000
```

```
[리밸런싱이 발생하면 벌어지는 일]

1. 컨슈머 1대가 리밸런싱에 걸림
2. 해당 컨슈머의 파티션이 다른 컨슈머에 재배정
3. 재배정 중에는 해당 파티션의 메시지 소비 정지
4. 10,000 msgs/sec가 계속 유입 → 브로커에 적체
5. 브로커 페이지 캐시 0.2초만에 밀림 → 디스크 읽기 발생
6. 전체 지연 폭증

→ 리밸런싱은 반드시 피해야 한다.
```

`max.poll.interval.ms=300000`(5분)은 기본값이며, max.poll.records=500에서 처리 시간이 이를 초과할 가능성은 거의 없다.

```
메시지 500개 × 0.01초 = 5초 (순차 처리 기준)
→ 300초(5분) 대비 충분한 여유
```

### Consumer 설정 요약

```properties
# CPU 제어: 100% → 50%로 감소
max.poll.records=500

# 네트워크 최소화: 최소 1MB 쌓인 후 응답
fetch.min.bytes=1048576
fetch.max.wait.ms=200

# 메모리 보호: fetch 크기 제한
fetch.max.bytes=104857600
max.partition.fetch.bytes=10485760

# 리밸런싱 방지
session.timeout.ms=30000
heartbeat.interval.ms=10000
max.poll.interval.ms=300000
```

---

## 전체 설정 한눈에 보기

### Topic

```properties
num.partitions=20
```

### Producer

```properties
compression.type=lz4
batch.size=5242880
linger.ms=5
max.request.size=10485760
buffer.memory=134217728
acks=1
```

### Broker

```properties
KAFKA_HEAP_OPTS="-Xmx1g -Xms1g"

message.max.bytes=5242880
replica.fetch.max.bytes=5242880
queued.max.requests=100
num.network.threads=8
num.io.threads=16
log.retention.hours=24
```

### Consumer (× 20대)

```properties
max.poll.records=500
fetch.min.bytes=1048576
fetch.max.wait.ms=200
fetch.max.bytes=104857600
max.partition.fetch.bytes=10485760
session.timeout.ms=30000
heartbeat.interval.ms=10000
max.poll.interval.ms=300000
```

### 튜닝 전후 비교

| 지표 | 튜닝 전 | 튜닝 후 |
|------|---------|---------|
| 네트워크 데이터량 | 10GB/sec | 5GB/sec (lz4 압축) |
| 네트워크 요청 수 | 10,000 req/sec | 2,000 req/sec (배치) |
| 브로커 인메모리 큐 | 최대 1,250MB (OOM 위험) | 최대 250MB (안전) |
| 컨슈머 CPU | 100% (1대) | 50% (× 20대) |
| 페이지 캐시 체류 시간 | 0.1초 | 0.2초 |

---

## 마무리 — 옵션이 아니라 병목을 읽어라

튜닝의 본질은 **"이 옵션을 어떤 값으로 바꿀까"가 아니라 "내 시스템의 병목이 어디인가"를 묻는 것**이다.

이 글에서 다룬 조건의 병목은 세 가지였다.

```
병목 1: 10GB/sec가 2GB 메모리를 관통한다
  → 압축(lz4)으로 데이터량 50% 절감
  → queued.max.requests=100으로 인메모리 요청 제한

병목 2: 컨슈머 CPU가 100%까지 치솟는다
  → max.poll.records=500으로 한 번에 처리하는 양을 절반으로 줄임
  → 컨슈머 20대로 부하 분산

병목 3: 네트워크 요청이 초당 10,000회다
  → batch.size=5MB로 요청 수 80% 감소
  → fetch.min.bytes=1MB로 빈 응답 제거
```

설정값은 조건이 바뀌면 바뀌어야 한다. 메시지 크기가 1MB가 아니라 1KB였다면 batch.size는 16KB로도 충분했을 것이다. 브로커 메모리가 32GB였다면 queued.max.requests를 줄일 이유가 없었을 것이다.

**하지만 "병목을 찾고, 그 병목을 풀어주는 옵션을 조정한다"는 사고 과정은 어떤 조건에서든 동일하다.**

Kafka 공식 문서에는 수백 개의 옵션이 있다. 전부 외울 필요 없다. **조건에서 숫자를 읽고, 숫자에서 병목을 찾고, 병목에서 옵션을 도출하면 된다.** 그것이 튜닝의 전부다.
