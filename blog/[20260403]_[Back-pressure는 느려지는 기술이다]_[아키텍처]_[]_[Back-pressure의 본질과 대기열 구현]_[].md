# Back-pressure는 느려지는 기술이다 — 빠르게 만드는 것만이 엔지니어링이 아닌 이유

## 들어가며

블랙 프라이데이에 주문이 100배 폭증하면, 서버를 100배 늘리면 해결될까?

직감적으로는 그렇다. 하지만 현실은 다르다. 서버를 10배 늘려도 DB 커넥션 풀은 50개이고, PG사의 TPS 제한은 바뀌지 않는다. 트래픽의 피크가 행사 시작 직후 10초에 몰린다면, 오토스케일링이 반응하기도 전에 시스템은 이미 터져 있다.

"더 빠르게, 더 많이 처리하자"는 엔지니어링의 본능이다. 그런데 이 본능이 해결하지 못하는 순간이 있다. **하류 시스템이 감당할 수 있는 속도 이상으로 상류가 밀어붙이면, 아무리 빨라도 터진다.**

이 글에서는 그 순간에 필요한 사고방식을 다룬다.

1. "더 빠르게"가 해결책이 되지 못하는 구조적 이유
2. Back-pressure라는 개념이 의미하는 것
3. 이 개념이 대기열 외에도 소프트웨어 전반에서 반복되는 이유
4. 실제 코드에서 "적절히 느리게"가 어떻게 구현되는가

---

## "더 빠르게"의 한계 — 스케일업과 아웃이 안 되는 이유

평소 초당 100건이던 주문 요청이 초당 10,000건으로 폭증한다고 하자.

```
[10,000명 동시 접속]
     └── POST /orders
           ├── 재고 확인 & 차감
           ├── 결제 처리
           └── 주문 저장
           → DB 커넥션 풀 고갈
           → 응답 지연 → 타임아웃
           → 전체 시스템 장애
```

첫 번째 반응은 서버를 늘리는 것이다. 합리적이다. 하지만 숫자를 들여다보면 한계가 보인다.

### 스케일 아웃의 병목

```
WAS 인스턴스: 10대로 확장
인스턴스당 DB 커넥션 풀: 50개
→ 총 DB 커넥션: 500개

주문 1건 처리 시간: 200ms (재고 + 결제 + 저장)
→ 이론적 최대 TPS: 500 / 0.2 = 2,500 TPS

요청: 10,000 TPS
→ 부족분: 7,500 TPS
→ 나머지는? 타임아웃.
```

DB 커넥션은 무한히 늘릴 수 없다. MySQL의 `max_connections` 기본값은 151이다. 1,000으로 올려도 커넥션 하나당 메모리를 잡아먹고, 컨텍스트 스위칭 비용이 커져서 오히려 처리량이 떨어진다. **WAS는 수평 확장이 쉽지만, DB와 PG는 그렇지 않다.**

### 오토스케일링의 시간차

```
t=0s   행사 시작. 트래픽 100 → 10,000 TPS (100배 폭증)
t=3s   모니터링 지표 수집
t=10s  오토스케일링 트리거
t=30s  새 인스턴스 부팅 시작
t=60s  헬스체크 통과, 트래픽 수신 시작

→ 트래픽 피크는 t=0~10s에 집중
→ 오토스케일링이 반응할 때는 이미 끝난 뒤
```

트래픽의 피크가 극단적으로 짧고 높은 경우, 오토스케일링은 **이미 끝난 불에 소방차를 보내는 것**과 같다. 물론 피크 이후에도 평소보다 높은 트래픽이 지속된다면 오토스케일링은 여전히 유효하다. 하지만 **피크 순간**을 버티는 건 오토스케일링의 몫이 아니다.

정리하면 두 가지다.

1. **수직적 한계**: DB, PG 같은 하류 시스템은 스케일이 제한적이다
2. **시간적 한계**: 피크가 인프라 반응 속도보다 빠르다

"더 빠르게"로는 이 두 가지를 넘을 수 없다. 그렇다면 다른 질문을 해야 한다.

---

## 느려지는 것이 답인 순간 — Back-pressure의 본질

질문을 바꿔보자.

> 시스템이 감당할 수 있는 속도는 얼마인가? 그리고 그 속도에 맞춰 요청을 조절할 수 있는가?

이것이 **Back-pressure**다.

Back-pressure는 하류 시스템이 감당할 수 있는 속도만큼만 상류의 요청을 흘려보내는 것이다. 수도관에 비유하면, 하류 파이프가 좁으면 상류에서 물을 천천히 틀어야 하는 것과 같다.

```
[Back-pressure 없이]

상류(유저 요청) ━━━━━━━━━━━━━━━━━━━━━━ 10,000 TPS
                         ↓
하류(DB)        ━━━━━━━━ 2,500 TPS (한계)
                         ↓
              차이 7,500 TPS → 타임아웃, 커넥션 고갈, 장애


[Back-pressure 적용]

상류(유저 요청) ━━━━━━━━━━━━━━━━━━━━━━ 10,000 TPS
                         ↓
              ┌─────── 대기열 ───────┐
              │  순서대로 보관        │
              │  "512번째, 약 3초"    │
              └──────────────────────┘
                         ↓
하류(DB)        ━━━━━━━━ 175 TPS (안전 마진 포함)
```

핵심은 간단하다. **처리할 수 없는 요청을 거부하거나 무시하는 대신, 보관하고 속도를 조절하는 것.** 시스템 입장에서는 안정적으로 처리하고, 유저 입장에서는 기다려서라도 결과를 얻을 수 있다.

### 175 TPS는 어디서 나왔는가

이 숫자는 직감이 아니라 시스템 조건에서 도출된다. 앞서 10대의 WAS가 총 500개의 DB 커넥션을 사용하는 시나리오를 봤다. 하지만 그 500개 전부를 주문 처리에 쓸 수는 없다. 상품 조회, 쿠폰 조회, 재고 확인 등 다른 API도 커넥션을 점유하기 때문이다. 여기서는 **주문 처리 전용으로 확보 가능한 커넥션을 50개**로 가정한다.

```
주문 처리 전용 DB 커넥션: 50 (전체 500 중 주문 할당분)
주문 1건 평균 처리 시간: 200ms

이론적 최대 TPS: 50 / 0.2 = 250 TPS
안전 마진 70%: 250 × 0.7 = 175 TPS
```

안전 마진을 70%로 잡는 이유는, GC나 네트워크 지터 같은 런타임 변수를 흡수해야 하기 때문이다. **평균이 아니라 피크를 견딜 수 있는 여유**가 있어야 한다.

---

## 반복되는 패턴 — Back-pressure는 대기열만의 것이 아니다

Back-pressure가 흥미로운 건, 이 개념이 소프트웨어의 여러 곳에서 **같은 형태로** 반복된다는 점이다. 이름은 다르지만, "하류가 감당할 수 있는 속도로 상류를 조절한다"는 본질은 같다.

### TCP Flow Control

```
송신자 → 수신자

수신자: "내 버퍼에 빈 공간이 16KB야" (Window Size)
송신자: "그러면 16KB만 보낼게"
수신자: "4KB 처리했어. 빈 공간 20KB" (Window Update)
송신자: "알겠어. 20KB까지 보낼게"

→ 수신자의 처리 속도에 맞춰 송신자가 전송량을 조절한다.
```

수십 년 전부터 TCP가 해온 것이 바로 Back-pressure다. 수신자의 버퍼가 감당할 수 있는 만큼만 보낸다.

### Reactive Streams

```
Publisher → Subscriber

Subscriber: "나 10개만 처리할 수 있어" (request(10))
Publisher: "알겠어. 10개만 보낼게"
Subscriber: "5개 처리 완료. 5개 더 줘" (request(5))
Publisher: "5개 보낸다"

→ Subscriber가 자기 처리 속도에 맞게 데이터를 요청한다.
```

Reactive Streams의 `request(n)` 메커니즘도 같은 원리다. "내가 처리할 수 있는 만큼만 달라"는 것.

### Kafka Consumer

```
Consumer: max.poll.records=500

브로커에 10,000개의 메시지가 쌓여 있어도,
한 번의 poll()에 500개만 가져온다.

→ Consumer의 CPU가 감당할 수 있는 양만 가져와서 처리한다.
```

Kafka Consumer의 `max.poll.records`도 Back-pressure다. 브로커에 아무리 메시지가 쌓여 있어도, Consumer가 한 번에 처리할 수 있는 양만 가져온다.

### Circuit Breaker

```
[하류(PG)가 느려지면]

정상: 요청 → PG → 200ms 응답
장애: 요청 → PG → 2000ms 응답... → 타임아웃

CircuitBreaker OPEN:
  요청 → 차단 → 즉시 실패 → PG에 요청을 보내지 않음

→ 하류가 감당 못 하면 상류에서 아예 보내지 않는다.
```

CircuitBreaker는 엄밀히 말하면 Back-pressure와 방식이 다르다. Back-pressure는 요청을 보관하고 속도를 조절하지만, CircuitBreaker는 요청을 즉시 실패시킨다. 하지만 **"하류가 감당 못 하면 상류에서 흐름을 끊는다"**는 목적은 같다. 속도를 조절하는 대신 아예 보내지 않는 — 가장 극단적인 형태의 하류 보호 전략이다.

이 네 가지를 나란히 놓으면 패턴이 보인다.

| 기술 | 상류 | 하류 | 조절 방식 |
|------|------|------|-----------|
| TCP Flow Control | 송신자 | 수신자 | Window Size로 전송량 제한 |
| Reactive Streams | Publisher | Subscriber | `request(n)`으로 요청량 제한 |
| Kafka Consumer | Broker | Consumer | `max.poll.records`로 fetch량 제한 |
| Circuit Breaker | 클라이언트 | 외부 API | 실패율 기반으로 호출 차단 |
| **대기열** | **유저** | **DB/PG** | **스케줄러로 초당 입장 수 제한** |

**"하류의 한계를 인정하고, 상류를 조절한다."** 이름만 다를 뿐, 전부 같은 이야기다.

---

## 대기열 — Back-pressure를 코드로 구현하기

Back-pressure의 개념은 이해했다. 그러면 주문 시스템에서 이걸 어떻게 구현할 수 있을까?

핵심 구성 요소는 세 가지다.

```
[유저] → 대기열 진입 (POST /queue/enter)
      → Redis Sorted Set에 userId + timestamp 저장
      → 순번 응답 (e.g. "512번째, 약 3초")

[스케줄러] → 50ms마다 실행
         → ZPOPMIN으로 9명 꺼내기
         → 입장 토큰 발급 (Redis SET + TTL 5분)

[유저] → 토큰 수신
      → POST /orders (Header: X-Entry-Token)
      → 토큰 검증 → 주문 처리 → 토큰 삭제
```

| 구성 요소 | 역할 | 조절하는 것 |
|-----------|------|-------------|
| **대기열** | 요청을 순서대로 보관 | 유입량 (아무리 많아도 보관) |
| **스케줄러** | 일정 주기로 N명씩 토큰 발급 | **처리 속도 (TPS)** |
| **입장 토큰** | 주문 API 진입 권한 | 동시 접근 수 |

Back-pressure의 핵심인 "속도 조절"은 **스케줄러**가 담당한다. 대기열에 10,000명이 있든 100,000명이 있든, 스케줄러는 50ms마다 9명만 꺼낸다. 이것이 하류(DB)를 보호하는 밸브다.

### 왜 Redis Sorted Set인가

대기열의 핵심 요구사항은 **순서 보장**, **중복 방지**, **빠른 순번 조회**다. Redis Sorted Set은 이 세 가지를 한 번에 해결한다.

```
ZADD  order-queue  {timestamp}  {userId}    // 대기열 진입 (NX: 중복 방지)
ZRANK order-queue  {userId}                 // 내 순번 조회 (0-based, O(log N))
ZCARD order-queue                           // 전체 대기 인원 (O(1))
ZPOPMIN order-queue {N}                     // 앞에서 N명 꺼내기 (스케줄러)
```

- **score = 진입 시각(timestamp)**: 먼저 들어온 사람이 앞 순번
- **member = userId**: Set 특성으로 중복 진입 자동 방지
- **인메모리**: 순번 조회가 μs 단위. 10,000명이 2초마다 Polling해도 감당 가능

### 도메인 레이어 — "속도를 조절하겠다"는 선언

```kotlin
interface OrderQueueRepository {
    fun enqueue(userId: Long, score: Double): Boolean
    fun getPosition(userId: Long): Long?
    fun getTotalSize(): Long
    fun popFront(count: Long): List<Long>
}
```

이 인터페이스는 도메인 레이어에 있다. Redis도 Spring도 모른다. 그런데 `popFront(count)` — **한 번에 몇 명을 꺼낼 것인가** — 가 파라미터로 드러나 있다. 이것이 Back-pressure의 핵심인 "속도 조절"을 타입 레벨에서 표현한 것이다.

### 인프라 레이어 — Redis로 구현

```kotlin
@Component
class OrderQueueRedisRepository(
    @Qualifier(REDIS_TEMPLATE_MASTER)
    private val masterRedisTemplate: RedisTemplate<String, String>,
    private val readRedisTemplate: RedisTemplate<String, String>,
) : OrderQueueRepository {

    private val masterZSet = masterRedisTemplate.opsForZSet()
    private val readZSet = readRedisTemplate.opsForZSet()

    override fun enqueue(userId: Long, score: Double): Boolean {
        // addIfAbsent = ZADD NX: 이미 존재하면 무시 (중복 진입 방지)
        return masterZSet.addIfAbsent(RedisKeys.orderQueueKey(), userId.toString(), score)
            ?: false
    }

    override fun getPosition(userId: Long): Long? {
        // ZRANK: 0-based 순번. 읽기 전용 Replica에서 조회
        return readZSet.rank(RedisKeys.orderQueueKey(), userId.toString())
    }

    override fun popFront(count: Long): List<Long> {
        // ZPOPMIN: score가 가장 낮은(= 가장 먼저 들어온) N명을 원자적으로 꺼낸다
        val result = masterZSet.popMin(RedisKeys.orderQueueKey(), count) ?: emptySet()
        return result.mapNotNull { it.value?.toLongOrNull() }
    }
}
```

`enqueue`에 `addIfAbsent`를 쓴 이유가 있다. 유저가 브라우저를 여러 개 열어서 중복 진입을 시도해도, Sorted Set의 Set 특성으로 자연스럽게 방지된다. 별도의 중복 체크 로직이 필요 없다.

읽기(순번 조회)는 `readRedisTemplate`(Replica), 쓰기(진입/꺼내기)는 `masterRedisTemplate`(Master)를 분리한 것도 눈여겨볼 만하다. 대기 인원이 10,000명이고 2초마다 Polling하면 초당 5,000건의 읽기 요청이 발생하는데, 이걸 Master에 보내면 쓰기와 경합한다.

---

## 스케줄러 — "얼마나 느리게"를 결정하는 코드

대기열은 요청을 보관하는 곳이고, 실제로 Back-pressure를 행사하는 건 스케줄러다. **"초당 몇 명을 통과시킬 것인가"를 결정하는 코드**이기 때문이다.

```kotlin
@Component
class QueueAdmissionScheduler(
    private val orderQueueService: OrderQueueService,
    private val queueFacade: QueueFacade,
    @Value("\${queue.admission.batch-size}") private val batchSize: Long,
    @Value("\${queue.admission.fixed-rate}") private val fixedRate: Long,
    @Value("\${queue.admission.jitter-range}") private val jitterRange: Long,
) : SmartLifecycle {

    private val executor: ScheduledExecutorService = Executors.newSingleThreadScheduledExecutor()

    @Volatile
    private var running = false

    fun admitUsers() {
        try {
            val admittedUserIds = orderQueueService.admitUsers(batchSize)
            if (admittedUserIds.isNotEmpty()) {
                log.info("대기열 입장 허용: {}명", admittedUserIds.size)
            }
            queueFacade.broadcastPositions(admittedUserIds) // SSE 또는 WebSocket으로 대기 순번 실시간 전달
        } catch (e: Exception) {
            log.warn("대기열 입장 허용 중 오류 발생", e)
        } finally {
            scheduleNext()
        }
    }

    fun calculateNextDelay(): Long {
        if (jitterRange <= 0) return fixedRate
        val jitter = ThreadLocalRandom.current().nextLong(-jitterRange, jitterRange + 1)
        return fixedRate + jitter
    }

    private fun scheduleNext() {
        if (running) {
            executor.schedule(::admitUsers, calculateNextDelay(), TimeUnit.MILLISECONDS)
        }
    }
}
```

### 설정값의 도출

```yaml
queue:
  admission:
    batch-size: 9
    fixed-rate: 50
    jitter-range: 20
```

이 세 숫자는 아까 도출한 175 TPS에서 나온다.

```
목표 TPS: 175
스케줄러 주기: 50ms (1초에 20회 실행)
배치 크기: 175 / 20 = 8.75 → 올림 9

→ 50ms마다 9명씩 토큰 발급
→ 초당 약 180명 입장 ≈ 175 TPS
```

50ms마다 9명. 이 숫자가 DB 커넥션 풀 50개, 주문 처리 200ms라는 시스템 조건에서 도출된 것이다. **하류의 한계가 상류의 속도를 결정한다.** 이것이 코드로 표현된 Back-pressure다.

### Jitter — 같은 시각에 몰리지 않게

`jitter-range: 20`은 스케줄러 실행 간격에 ±20ms의 랜덤 편차를 준다.

```
jitter 없이: 정확히 50ms → 50ms → 50ms → ...
jitter 적용: 42ms → 67ms → 33ms → 55ms → ...
```

왜 이게 필요한가? 스케줄러가 정확히 50ms마다 실행되면, 토큰을 받은 9명이 **거의 동시에** 주문 API를 호출한다. 이게 반복되면 50ms 주기의 미니 스파이크가 계속 발생한다. Jitter는 이 주기를 흐트러뜨려서 부하를 자연스럽게 분산시킨다.

이건 이 프로젝트만의 특수한 기법이 아니다. AWS의 공식 문서에서도 분산 시스템의 재시도 전략에 Jitter를 권장한다. **규칙적인 간격은 동기화를 만들고, 동기화는 동시 부하를 만든다.**

---

## 토큰 — "통과 허가증"이 필요한 이유

대기열에서 빠져나온 유저에게 바로 주문을 허용하면 안 될까? 왜 토큰이라는 중간 단계가 필요한가?

### 토큰 없이 설계하면

```
스케줄러: "512번 유저, 이제 주문해도 됩니다"
유저: (브라우저를 닫고 갔다)
→ 자리만 차지. 다음 유저도 대기 중.
→ 시간이 지나도 아무도 주문하지 않는 빈 슬롯 발생
```

토큰은 이 문제를 TTL로 해결한다.

```kotlin
fun admitUsers(batchSize: Long): List<Long> {
    val userIds = orderQueueRepository.popFront(batchSize)
    userIds.forEach { userId ->
        val token = UUID.randomUUID().toString()
        entryTokenRepository.issue(userId, token, ENTRY_TOKEN_TTL_SECONDS) // 300초 TTL
    }
    return userIds
}
```

```
SET  entry-token:{userId}  {UUID}  EX 300    // 5분 후 자동 만료
```

토큰이 5분 안에 사용되지 않으면 자동으로 사라진다. 이탈한 유저의 자리는 자연스럽게 회수되고, 그만큼 다음 유저에게 기회가 돌아간다.

주문 API에서는 토큰을 검증하고 소비한다.

```kotlin
fun validateAndConsumeToken(userId: Long, token: String) {
    val storedToken = entryTokenRepository.get(userId)
        ?: throw CoreException(ErrorType.FORBIDDEN, "입장 토큰이 존재하지 않습니다.")

    if (storedToken == EntryTokenRepository.BYPASS_TOKEN) {
        log.warn("대기열 bypass 모드: 토큰 검증 스킵. userId={}", userId)
        return
    }

    if (storedToken != token) {
        throw CoreException(ErrorType.FORBIDDEN, "입장 토큰이 일치하지 않습니다.")
    }

    entryTokenRepository.consume(userId) // DEL entry-token:{userId}
}
```

토큰이 없으면 403. 토큰이 틀리면 403. 토큰이 맞으면 주문 처리 후 삭제. **한 번 쓰면 사라지는 일회용 통과 허가증**이다.

> **주의**: 위 코드는 `get`과 `consume` 사이에 시간차가 있어, 같은 유저가 동일 토큰으로 동시에 두 번 요청하면 둘 다 통과할 수 있는 Race Condition이 존재한다. 프로덕션에서는 Redis Lua 스크립트로 조회 + 비교 + 삭제를 원자적으로 처리하거나, 주문 서비스에서 멱등성 키(Idempotency Key)로 중복 주문을 방지하는 것이 안전하다.

여기서 `BYPASS_TOKEN`이 눈에 띈다. 이건 뒤에서 다룬다.

---

## Redis 장애 시 — "느려지는 기술"이 작동하지 않을 때

Back-pressure의 핵심 인프라인 Redis가 죽으면 어떻게 해야 할까?

이건 기술 문제가 아니라 **비즈니스 판단**이다. 선택지는 세 가지다.

| 전략 | 동작 | 트레이드오프 |
|------|------|-------------|
| 전면 차단 | 대기열 진입 자체를 막고 "잠시 후 다시 시도" | 안전하지만 서비스 중단 |
| 대기열 우회 | 대기열 없이 주문 API 직접 접근 허용 | 서비스 유지하지만 과부하 위험 |
| Fallback 큐 | 로컬 메모리 큐나 Kafka로 임시 전환 | 순번 정확성 떨어지지만 서비스 유지 |

이 프로젝트에서는 **대기열 우회(bypass)**를 선택했다. "Redis가 죽었을 때, 주문을 아예 막는 것보다는 과부하 위험을 감수하고 허용하는 것이 비즈니스적으로 낫다"는 판단이다.

이 판단이 코드에 드러나는 방식이 인상적이다.

```kotlin
// Redis 장애 시 대기열 진입 bypass
internal fun enqueueFallback(userId: Long, score: Double, e: Exception): Boolean {
    log.warn("대기열 진입 bypass: userId={}", userId, e)
    return true  // "대기열에 들어간 것으로 간주"
}

// Redis 장애 시 토큰 조회 bypass
internal fun getFallback(userId: Long, e: Exception): String? {
    log.warn("토큰 조회 bypass: userId={}", userId, e)
    return EntryTokenRepository.BYPASS_TOKEN  // 특수 토큰 반환
}
```

`enqueueFallback`은 `true`를 반환한다. "대기열 진입 성공"이라고 응답하는 것이다. 실제로 Redis에 저장되지 않았지만, 유저 입장에서는 정상 흐름을 탄다.

`getFallback`은 `BYPASS_TOKEN`을 반환한다. 앞서 본 `validateAndConsumeToken`에서 이 토큰을 만나면 검증을 건너뛴다. **Redis 없이도 주문이 가능한 경로가 열리는 것**이다.

여기에 Bulkhead가 안전장치로 추가되어 있다.

```yaml
resilience4j:
  bulkhead:
    instances:
      order-place:
        max-concurrent-calls: 30   # bypass 시 동시 주문 제한
        max-wait-duration: 0       # 대기 없이 즉시 실패
```

bypass 모드에서 10,000명이 동시에 주문 API를 때리면 시스템이 터진다. Bulkhead는 동시 주문을 30건으로 제한해서 DB를 보호한다. **대기열이라는 1차 방어선이 무너져도, Bulkhead라는 2차 방어선이 버틴다.**

중요한 건 이 설계 결정이 코드를 짜기 전에 이루어졌다는 점이다. "Redis 장애 시 우리 서비스는 어떻게 동작해야 하는가?"라는 질문에 답을 정해둔 것이다. 장애가 발생한 뒤에 판단하면 늦다.

---

## 마무리 — 빠르게만 만들면 되는 줄 알았다

Back-pressure를 공부하기 전까지, 트래픽이 몰리면 "어떻게 하면 더 빠르게 처리할까?"만 생각했다. 서버를 늘리고, 캐시를 붙이고, 쿼리를 최적화하고. 전부 맞는 접근이다. 하지만 그것만으로는 해결되지 않는 순간이 있다.

DB 커넥션 풀은 유한하고, PG사의 TPS는 우리가 정하는 것이 아니고, 오토스케일링은 피크보다 느리다. **하류의 한계는 상류의 노력만으로 넘을 수 없다.** 이 사실을 인정하는 것이 Back-pressure의 출발점이다.

> **Back-pressure는 "느려지는 기술"이다**
> - 하류가 감당할 수 있는 속도만큼만 상류를 흘려보낸다
> - 시스템 조건(DB 커넥션 풀 50, 처리 시간 200ms)에서 안전한 TPS(175)를 도출한다
> - 스케줄러가 50ms마다 9명씩만 통과시킨다 — 이것이 속도 조절의 실체다

> **이 패턴은 대기열만의 것이 아니다**
> - TCP는 Window Size로, Reactive Streams는 `request(n)`으로, Kafka Consumer는 `max.poll.records`로 같은 일을 한다
> - Circuit Breaker는 Back-pressure와 방식은 다르지만 목적은 같다 — 가장 극단적인 형태의 하류 보호 전략이다
> - 이름은 달라도 본질은 하나다: **하류의 한계를 인정하고, 상류를 조절한다**

> **장애 대응은 코드 전에 합의한다**
> - Redis가 죽으면 전면 차단인가, 우회인가, Fallback인가?
> - 이 프로젝트는 bypass + Bulkhead를 선택했다 — 정답이 아니라 판단이다
> - "장애 시 어떻게 동작할 것인가?"를 정해두는 행위 자체가 핵심이다

"더 빠르게"가 엔지니어링의 전부인 줄 알았다. 그런데 **"적절히 느리게"도 엔지니어링이었다.** 하류의 한계를 인정하고, 그 한계에 맞춰 상류를 설계하는 것. 그게 Back-pressure의 본질이고, 시스템을 안정적으로 만드는 또 다른 방법이다.
