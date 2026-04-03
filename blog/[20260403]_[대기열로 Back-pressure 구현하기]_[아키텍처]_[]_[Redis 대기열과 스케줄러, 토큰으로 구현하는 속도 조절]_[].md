# 대기열로 Back-pressure 구현하기 — Redis, 스케줄러, 토큰

## 들어가며

이전 글에서 Back-pressure의 본질을 다뤘다. 하류가 감당할 수 있는 속도만큼만 상류를 흘려보내는 것. 이 글에서는 그 개념을 주문 시스템의 대기열로 구현하는 과정을 다룬다.

구현의 출발점은 시스템 조건에서 도출한 숫자다.

```
주문 처리 전용 DB 커넥션: 50
주문 1건 평균 처리 시간: 200ms
안전 마진 70% 적용 → 목표 TPS: 175
```

이 175 TPS를 넘지 않도록 유저의 주문 진입을 조절하는 것이 이 글의 목표다.

1. 대기열의 구조와 Redis Sorted Set 선택 이유
2. 스케줄러가 "얼마나 느리게"를 결정하는 방법
3. 토큰이라는 중간 단계가 필요한 이유
4. Redis 장애 시 대응 전략

---

## 대기열 — Back-pressure를 코드로 구현하기

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

이 세 숫자는 목표 175 TPS에서 나온다.

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

## 마무리

대기열, 스케줄러, 토큰. 세 가지 구성 요소가 각각의 역할을 나눠 가진다.

> **대기열은 보관하고, 스케줄러는 조절하고, 토큰은 통제한다**
> - 대기열(Redis Sorted Set)은 요청을 순서대로 보관한다
> - 스케줄러는 50ms마다 9명씩만 통과시킨다 — 이것이 속도 조절의 실체다
> - 토큰은 TTL로 이탈한 유저의 자리를 자연스럽게 회수한다

> **장애 대응은 코드 전에 합의한다**
> - Redis가 죽으면 전면 차단인가, 우회인가, Fallback인가?
> - 이 프로젝트는 bypass + Bulkhead를 선택했다 — 정답이 아니라 판단이다
> - "장애 시 어떻게 동작할 것인가?"를 정해두는 행위 자체가 핵심이다

Back-pressure는 개념으로는 단순하다. 하류가 감당할 수 있는 만큼만 흘려보내면 된다. 하지만 그 "만큼"을 코드로 옮기려면, 숫자를 도출하고, 구조를 나누고, 장애까지 설계해야 한다. **"적절히 느리게"는 생각보다 정교한 엔지니어링이다.**
