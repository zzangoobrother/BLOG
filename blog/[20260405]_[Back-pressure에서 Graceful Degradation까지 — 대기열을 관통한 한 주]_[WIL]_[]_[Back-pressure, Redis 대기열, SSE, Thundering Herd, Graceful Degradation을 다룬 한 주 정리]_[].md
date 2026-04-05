# WIL - Back-pressure에서 Graceful Degradation까지 — 대기열을 관통한 한 주

## 이번 주 요약

이번 주는 **"속도를 조절하는 시스템"을 처음부터 끝까지 설계하고 구현**한 한 주였다. Back-pressure의 본질을 글로 정리하면서 "더 빠르게"가 답이 아닌 순간을 이론으로 잡고, Redis Sorted Set 기반 대기열을 직접 구축하면서 코드로 증명했다. 대기열 진입부터 입장 토큰 발급, SSE 실시간 순번 Push, Thundering Herd 완화를 위한 Jitter 스케줄러까지 올렸다. 여기서 끝이 아니었다. Redis가 죽으면 대기열도 죽는다는 문제에 CircuitBreaker + Fallback으로 Graceful Degradation을 적용하고, Bypass 모드에서도 DB가 터지지 않도록 Bulkhead로 동시 주문을 제한했다. 마지막에는 토큰 검증을 Lua 스크립트 기반 원자적 연산으로 바꾸고, 토큰 발급 실패 시 사용자가 대기열에서 영구 이탈하는 버그까지 잡았다.

---

## 이번 주에 정리한 것

### Back-pressure는 느려지는 기술이다

"더 빠르게, 더 많이 처리하자"는 엔지니어링의 본능이다. 하지만 **하류 시스템이 감당할 수 있는 속도 이상으로 상류가 밀어붙이면, 아무리 빨라도 터진다.** 서버를 10배 늘려도 DB 커넥션 풀은 50개이고, PG사의 TPS 제한은 바뀌지 않는다.

- **스케일 아웃의 한계**: WAS 10대 × 커넥션 50개 = 총 500개. 주문 1건 200ms면 이론적 최대 2,500 TPS. 요청이 10,000 TPS면 7,500건은 타임아웃.
- **Back-pressure의 본질**: "빠르게 처리하는 것"이 아니라 **"하류가 감당할 수 있는 속도로 상류를 늦추는 것"**. 의도적으로 느려지는 기술이다.
- **대기열 외에도 반복되는 패턴**: TCP의 윈도우 크기 조절, Kafka Consumer의 `max.poll.records`, Reactor의 `onBackpressureBuffer`. 레벨만 다를 뿐 같은 원리다.

---

### 대기열로 Back-pressure 구현하기

Back-pressure의 개념을 주문 시스템의 대기열로 구현하는 과정을 다뤘다. 출발점은 **시스템 조건에서 도출한 숫자**다.

- **대기열**: 요청을 순서대로 보관. Redis Sorted Set으로 유입량을 흡수한다.
- **스케줄러**: 50ms마다 9명씩 토큰 발급. 이것이 하류(DB)를 보호하는 밸브다. Back-pressure의 핵심인 "속도 조절"을 담당한다.
- **입장 토큰**: 주문 API 진입 권한. TTL 5분으로 미사용 토큰을 자동 회수한다.

대기열에 10,000명이 있든 100,000명이 있든, 스케줄러는 50ms마다 9명만 꺼낸다. 목표 TPS 175를 넘지 않도록 유저의 주문 진입을 조절하는 것이 대기열의 전부다.

---

## 이번 주에 구현한 것

이번 주 커밋: 17건 / 70개+ 파일 변경 / +3,800줄

### 1. Redis Sorted Set 기반 주문 대기열

대기열의 기반 저장소를 구현했다.

| 구성 요소 | 설명 |
|----------|------|
| `OrderQueueRedisRepository` | Redis Sorted Set 기반. userId가 member, timestamp가 score. FIFO 순서 보장 |
| `OrderQueueService` | 대기열 진입, 순번 조회, 예상 대기 시간 계산. 초당 175명 기준으로 산출 |
| `QueuePosition` | 순번, 전체 크기, 예상 대기 시간을 담는 도메인 모델 |

핵심은 **Redis Sorted Set을 선택한 이유**다. 일반 List는 중간 위치 조회가 O(N)이지만, Sorted Set은 `ZRANK`로 O(log N)에 순번을 알 수 있다. 대기열에서 "내가 몇 번째인지" 물어보는 것이 가장 빈번한 연산이니, 이 선택이 자연스럽다.

### 2. 입장 토큰 저장소 및 대기열 서비스

스케줄러가 대기열에서 꺼낸 사용자에게 발급하는 **입장 토큰**을 구현했다.

| 구성 요소 | 설명 |
|----------|------|
| `EntryTokenRedisRepository` | Redis String으로 토큰 저장. TTL 300초. UUID 기반 |
| `QueueAdmissionScheduler` | 50ms마다 실행. 대기열 앞쪽에서 9명 pop → 토큰 발급 |

토큰이 필요한 이유는 명확하다. 대기열을 통과한 사용자만 주문 API에 접근할 수 있어야 한다. 토큰 없이 직접 주문 API를 호출하면 거부된다.

### 3. 대기열 진입/순번 조회 API 및 동적 Polling 주기

대기열 API와 클라이언트 Polling 전략을 구현했다.

| API | 설명 |
|-----|------|
| `POST /api/v1/queue/enter` | 대기열 진입. 현재 순번과 Polling 주기 응답 |
| `GET /api/v1/queue/position` | 순번 조회. 토큰 발급 여부, bypass 상태 포함 |

**동적 Polling 주기**가 핵심이다. 순번이 앞쪽이면 자주, 뒤쪽이면 드물게 조회한다.

| 순번 | Polling 주기 | 이유 |
|------|-------------|------|
| 1~10 | 1초 | 곧 입장. 즉각 반응 필요 |
| 11~50 | 2초 | 가까운 미래에 입장 |
| 51~200 | 3초 | 잠시 대기 |
| 200+ | 5초 | 오래 기다림. 서버 부하 줄이기 |

모든 클라이언트가 1초마다 Polling하면 대기열 자체가 Redis 부하를 만든다. **대기열이 보호해야 할 시스템을, 대기열이 공격하는 아이러니**를 막기 위한 설계다.

### 4. SSE 기반 실시간 순번 Push

Polling의 한계를 보완하기 위해 SSE(Server-Sent Events)를 추가했다.

| 구성 요소 | 설명 |
|----------|------|
| `QueueEmitterRepository` | 도메인 인터페이스. SSE 커넥션 관리 추상화 |
| `SseEmitterRepositoryImpl` | ConcurrentHashMap 기반 인메모리 구현. userId → SseEmitter 매핑 |
| `QueueStreamController` | `GET /api/v1/queue/stream`. 5분 타임아웃 SSE 커넥션 |
| `QueueFacade.broadcastPositions()` | 스케줄러가 토큰 발급 후 모든 대기 중인 사용자에게 순번 Push |

SSE 이벤트 타입을 세 가지로 분리했다.

| 이벤트 | 의미 | 클라이언트 행동 |
|--------|------|----------------|
| `position` | 현재 순번 업데이트 | 화면에 순번 표시 |
| `admitted` | 토큰 발급됨 | 주문 페이지로 이동 |
| `bypass` | 대기열 비활성화 | 바로 주문 가능 |

### 5. Thundering Herd 완화를 위한 Jitter 스케줄러

대기열이 해결한 문제의 **축소판**이 다시 등장했다. 스케줄러가 50ms마다 9명에게 토큰을 발급하면, 9명이 거의 동시에 주문 API를 호출한다.

| 문제 | 해결 |
|------|------|
| 50ms 고정 간격 → 동기화된 burst | ±20ms Jitter 적용 → [30ms, 70ms] 범위로 분산 |
| `ThreadLocalRandom` 사용 | 스레드 간 경합 없는 난수 생성 |
| `SmartLifecycle` 구현 | 스케줄러의 graceful startup/shutdown |

**문제를 해결하면 더 작은 문제가 생긴다.** 10,000명의 동시 요청을 대기열로 해결했더니, 9명의 동시 요청이라는 축소판이 등장했다. 완전한 해결이 아니라 **평탄화(smoothing)**라는 사실을 인식하고, Jitter로 burst를 분산시켰다.

### 6. Redis 장애 시 Graceful Degradation

대기열의 가장 큰 약점 — **Redis가 죽으면 대기열도 죽는다.** 이 문제를 CircuitBreaker + Retry + Fallback으로 해결했다.

| 구성 요소 | 역할 |
|----------|------|
| `@CircuitBreaker(name = "order-queue")` | 최근 10건 중 50% 이상 실패 시 30초간 호출 차단 |
| `@Retry` | 일시적 장애 시 재시도 |
| Fallback 메서드 | CircuitBreaker OPEN 시 대기열 없이 진입 허용 |
| `CircuitBreakerQueueHealthChecker` | CircuitBreaker 상태로 bypass 여부 판단 |

**Graceful Degradation의 전체 흐름:**

```
정상 (CLOSED)
  → Redis 연속 실패
  → CircuitBreaker OPEN
  → isBypassed() = true
  → 대기열 없이 주문 허용 (bypass 모드)
  → 30초 후 HALF_OPEN
  → Redis 3회 성공
  → CLOSED 복귀 → 정상 대기열 재개
```

Fallback 전략을 설계할 때 가장 고민한 부분은, **Redis 장애 시 사용자를 차단할 것인가 통과시킬 것인가**였다. 대기열은 보호 장치이지 필수 기능이 아니다. Redis가 죽었다고 주문을 못 하게 하면, 보호 장치가 오히려 서비스를 죽이는 셈이다. 그래서 bypass를 선택했다.

### 7. Bypass 모드 동시 주문 제한을 위한 Bulkhead

Bypass 모드는 대기열 없이 주문을 허용하므로, **DB와 결제 시스템에 부하가 직격**한다. Bulkhead로 동시 주문 수를 제한했다.

| 설정 | 값 | 이유 |
|------|-----|------|
| `max-concurrent-calls` | 30 | DB 커넥션 풀 대비 안전한 동시 처리량 |
| `max-wait-duration` | 0 | fail-fast. 대기하면 스레드가 묶인다 |

Bulkhead 초과 시 즉시 `SERVICE_UNAVAILABLE`을 반환한다. bypass 모드라도 시스템이 감당할 수 있는 만큼만 처리한다.

### 8. 리팩토링 — bypass와 토큰 검증 개선

| 변경 | 이전 | 이후 |
|------|------|------|
| bypass 판단 | 매직 토큰 문자열 비교 | `QueueHealthChecker.isBypassed()` 명시적 상태 체크 |
| 토큰 검증 | GET → 비교 → DEL (3 연산) | Lua 스크립트로 compare-and-delete (1 원자적 연산) |

Lua 스크립트의 핵심:

```lua
if redis.call('GET', KEYS[1]) == ARGV[1] then
    redis.call('DEL', KEYS[1])
    return 1
end
return 0
```

GET → 비교 → DEL이 분리되면, 두 요청이 같은 토큰으로 동시에 주문할 수 있다. Lua 스크립트는 Redis에서 **단일 연산으로 실행**되므로 이 경쟁 조건이 원천 차단된다.

### 9. 버그 수정 — 토큰 발급 실패 시 영구 이탈

스케줄러가 대기열에서 사용자를 `ZPOPMIN`으로 꺼낸 뒤 토큰 발급에 실패하면, **사용자가 대기열에도 없고 토큰도 없는 상태**에 빠졌다.

| 문제 | 수정 |
|------|------|
| `ZPOPMIN` 후 토큰 발급 실패 → 사용자 영구 이탈 | 토큰 발급 실패한 사용자를 score 0.0으로 `requeue` → 최우선 순위로 복귀 |

score를 0.0으로 설정하는 이유는, 이미 오래 기다린 사용자가 시스템 오류로 뒤로 밀리면 안 되기 때문이다.

### 10. 테스트

| 테스트 종류 | 내용 |
|------------|------|
| **단위 테스트** | OrderQueueService 순번 계산, QueuePosition 모델, Jitter 범위 검증 |
| **통합 테스트** | Redis 연동 대기열 enqueue/dequeue, EntryToken 발급/소비, CircuitBreaker Fallback 동작 |
| **동시성 테스트** | Lua 스크립트 기반 토큰의 원자적 소비 검증. 동일 토큰 동시 요청 시 하나만 성공 |
| **E2E 테스트** | 대기열 진입 → 순번 조회 → SSE 스트림 → 토큰 수신 → 주문 처리 전 과정 |
| **Bypass 테스트** | CircuitBreaker OPEN 시 bypass 모드 전환, Bulkhead 초과 시 거부 |
| **HTTP 요청 파일** | IntelliJ HTTP Client용 queue-v1.http 추가 |

---

## 이번 주의 깨달음

이번 주는 **"문제를 해결하면 더 작은 문제가 생긴다"**는 것을 단계적으로 체감한 한 주였다.

처음에는 단순했다. 10,000명이 동시에 주문하면 DB가 터진다. 대기열을 넣으면 된다. Redis Sorted Set으로 대기열을 만들고, 스케줄러로 175 TPS에 맞춰 토큰을 발급했다. 문제 해결.

그런데 바로 다음 문제가 등장했다. 스케줄러가 50ms마다 9명에게 토큰을 발급하면, 9명이 동시에 주문 API를 호출한다. Thundering Herd의 축소판이다. Jitter로 발급 간격을 흔들어서 완화했다.

그 다음 문제. 유저가 대기열에서 기다리는 동안 순번을 모르면 새로고침을 누르거나 이탈한다. Polling으로 순번을 알려주되, 모든 유저가 1초마다 Polling하면 Redis가 대기열 조회만으로 과부하된다. 동적 Polling 주기와 SSE를 조합해서 해결했다.

그 다음 문제. Redis가 죽으면? 대기열의 존재 이유는 DB 보호인데, 대기열이 죽었다고 서비스를 중단하면 보호 장치가 서비스를 죽이는 아이러니다. CircuitBreaker로 bypass 모드를 만들었다. 하지만 bypass 모드에서 모든 요청이 DB에 직격하면 원래 문제로 돌아간다. Bulkhead로 동시 주문을 30개로 제한했다.

토큰 검증에서도 경쟁 조건을 발견했다. GET → 비교 → DEL이 분리되면 동일 토큰으로 두 번 주문이 가능하다. Lua 스크립트로 원자적 compare-and-delete를 적용했다. 그리고 토큰 발급 실패 시 사용자가 대기열에서 영구 이탈하는 버그도 잡았다.

돌아보면, 대기열을 만드는 것은 시작에 불과했다. **진짜 싸움은 대기열을 만든 다음에 일어났다.** Thundering Herd, 유저 이탈, Redis 장애, 경쟁 조건, 토큰 유실 — 하나를 해결하면 다음 문제가 등장하고, 그 문제를 해결하면 또 다음 문제가 나타났다.

결국 이번 주에 배운 건 하나다 — **시스템 설계는 "문제를 해결하는 것"이 아니라 "문제의 크기를 줄여나가는 것"이다.** 10,000명의 동시 요청을 175 TPS로, Thundering Herd를 Jitter로, Redis 장애를 bypass + Bulkhead로. 문제를 없앤 게 아니라, 시스템이 감당할 수 있는 크기로 줄인 것이다.
