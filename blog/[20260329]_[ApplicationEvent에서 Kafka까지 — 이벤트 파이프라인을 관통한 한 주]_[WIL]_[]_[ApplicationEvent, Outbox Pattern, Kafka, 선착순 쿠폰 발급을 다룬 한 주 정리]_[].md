# WIL - ApplicationEvent에서 Kafka까지 — 이벤트 파이프라인을 관통한 한 주

## 이번 주 요약

이번 주는 **이벤트 기반 아키텍처를 인프로세스에서 분산 시스템까지 확장**한 한 주였다. 좋아요 집계를 ApplicationEvent로 분리하는 것에서 시작해, Transactional Outbox Pattern으로 이벤트의 신뢰성을 확보하고, Kafka Producer-Consumer 파이프라인을 구축해 commerce-streamer로 집계를 넘겼다. 마지막에는 Redis Lua 스크립트와 Kafka를 조합한 선착순 쿠폰 발급까지 구현했다. Kafka 설정의 본질과 튜닝 사고법을 글로 정리하면서 이론을 잡고, 프로젝트에 직접 적용하면서 체감하는 순환이 이번 주에도 이어졌다.

---

## 이번 주에 정리한 것

### Kafka 설정의 본질 — Producer, Broker, Consumer

Kafka 설정을 프로퍼티 목록으로 외우는 대신, **"이 설정이 어떤 질문에 대한 답인가"**라는 관점으로 정리한 글이었다.

- **Producer**: `acks`는 "메시지를 보냈다"의 기준을 정하고, `retries` + `enable.idempotence`는 "실패하면 어떻게 할 것인가"에 답한다. `batch.size`와 `linger.ms`는 지연과 처리량 사이의 트레이드오프를 조절한다.
- **Broker**: `replication.factor`와 `min.insync.replicas`가 메시지 내구성을 결정한다. `acks=all`이라도 ISR이 1이면 의미가 없다는 것을 다뤘다.
- **Consumer**: `enable.auto.commit`은 "처리 완료"의 정의를 바꾸고, `max.poll.records`와 `max.poll.interval.ms`는 리밸런싱과 처리량의 균형을 결정한다.

결국 수십 개의 프로퍼티가 **신뢰성, 성능, 장애 대응** 세 가지 질문으로 수렴한다는 것을 정리했다.

---

### Kafka 튜닝의 본질 — 조건이 옵션을 결정한다

설정의 의미를 아는 것과 튜닝을 할 수 있는 것은 다르다. 이 글에서는 **구체적인 시스템 조건이 주어졌을 때, 옵션을 도출하는 사고 과정**을 다뤘다.

- **숫자부터 읽는다**: 초당 데이터량, 브로커 메모리, 컨슈머 처리 속도에서 병목을 도출한다.
- **세 목표의 교집합**: 압축은 네트워크를 줄이지만 CPU를 잡아먹고, 배치는 네트워크를 줄이지만 메모리를 잡아먹고, 병렬화는 CPU를 줄이지만 네트워크를 늘린다. 어느 하나를 극단적으로 밀면 다른 하나가 터진다.
- **레시피가 아닌 도출**: "batch.size는 이렇게"가 아니라 **"이 조건에서는 이 값이어야 하는 이유"**에 집중한다.

---

## 이번 주에 구현한 것

### 1. ApplicationEvent 기반 이벤트 분리

좋아요 집계와 주문-결제 부가 로직을 메인 트랜잭션에서 분리했다.

| 구성 요소 | 설명 |
|----------|------|
| `ProductEventHandler` | 좋아요/좋아요 취소 이벤트를 받아 likeCount를 비동기 갱신. Eventual Consistency 기반 |
| `OrderEventHandler` | 주문 완료 이벤트를 받아 결제 후속 처리 실행 |
| `UserActionLogHandler` | 상품 조회, 좋아요, 주문 등 유저 행동을 비동기 로깅 |
| `AsyncConfig` | ThreadPoolTaskExecutor 설정으로 비동기 이벤트 처리 스레드 풀 관리 |

핵심은 **관심사 분리**다. 좋아요 API의 책임은 "좋아요를 토글하는 것"이지 "집계를 갱신하는 것"이 아니다. 부가 로직을 이벤트로 밀어내자, 메인 트랜잭션이 가벼워지고 각 핸들러의 테스트가 독립적으로 가능해졌다.

### 2. 좋아요 동시성 처리 개선 — 비관적 락에서 INSERT IGNORE로

기존에는 좋아요 중복 방지를 비관적 락으로 처리했다. 동작은 하지만, 좋아요처럼 단순한 토글에 행 잠금은 과했다.

- **변경 전**: `SELECT ... FOR UPDATE`로 행을 잠그고, 중복 여부 확인 후 INSERT
- **변경 후**: `INSERT IGNORE`로 DB의 유니크 제약 조건에 중복 방지를 위임

락을 잡지 않으니 동시 요청 시 대기가 사라졌다. **"동시성 문제를 코드로 풀지 말고, DB가 이미 보장하는 것에 기대라"**는 이전 주의 깨달음을 적용한 케이스다.

### 3. Kafka 이벤트 파이프라인 — Outbox Pattern에서 Consumer까지

인프로세스 ApplicationEvent를 넘어, **시스템 간 이벤트 전파**를 Kafka로 구축했다.

| 단계 | 구현 | 역할 |
|------|------|------|
| **이벤트 스키마** | `EventEnvelope`, `CatalogPayloads`, `OrderPayloads` | 이벤트 타입별 페이로드를 표준화 |
| **Outbox Pattern** | `OutboxEvent`, `OutboxEventPublisher`, `OutboxRelayScheduler` | 도메인 이벤트를 DB에 먼저 저장하고, 스케줄러가 Kafka로 릴레이. 트랜잭션과 메시지 발행의 원자성 보장 |
| **Consumer (commerce-streamer)** | `CatalogEventConsumer`, `OrderEventConsumer` | 카탈로그/주문 이벤트를 소비하여 product_metrics 집계 |
| **집계** | `ProductMetrics`, `ProductMetricsRepository` | 조회수, 좋아요수, 주문수를 상품별로 집계 |
| **DLQ** | `KafkaErrorHandlerConfig` | 처리 실패 메시지를 Dead Letter Queue로 라우팅 |

Outbox Pattern을 선택한 이유는 명확하다. ApplicationEvent로 Kafka Producer를 호출하면, **DB 커밋은 성공했는데 Kafka 발행이 실패**하는 시나리오가 생긴다. Outbox에 먼저 저장하고 별도 스케줄러가 릴레이하면, 최소 한 번 전달(at-least-once)이 보장된다.

### 4. 선착순 쿠폰 발급 — Redis Lua + Kafka

선착순 쿠폰 발급을 **Redis로 빠르게 검증하고, Kafka로 비동기 처리**하는 구조로 구현했다.

| Phase | 구현 | 설명 |
|-------|------|------|
| **Phase 1** | `CouponIssueRequest`, `CouponIssueStatus` | 쿠폰 발급 요청 도메인 모델. REQUESTED → ISSUED / FAILED 상태 전이 |
| **Phase 2** | `CouponIssueAsyncController`, Redis Lua 스크립트 | API 요청 시 Redis Lua로 원자적 재고 차감 + 중복 방지. 통과하면 Kafka로 발급 요청 전송 |
| **Phase 3** | `CouponIssueConsumer`, `CouponIssueFacade` | Kafka Consumer가 발급 요청을 받아 실제 쿠폰 발급 처리 |
| **Phase 4** | E2E 테스트, 통합 테스트 | 선착순 시나리오별 전체 흐름 검증 |

Redis Lua 스크립트를 사용한 이유는, **재고 확인 + 차감 + 중복 체크**를 하나의 원자적 연산으로 실행해야 하기 때문이다. 이 세 연산이 분리되면 선착순이 깨진다.

### 5. VIEWED 이벤트 DIP 적용 및 리팩토링

상품 조회 시 발생하는 VIEWED 이벤트를 Outbox를 거치지 않고 Kafka로 직접 발행하도록 변경했다.

- `DirectEventPublisher` 인터페이스를 도메인 레이어에 배치
- `DirectKafkaEventPublisher`가 인프라 레이어에서 구현
- 조회 이벤트처럼 유실되어도 치명적이지 않은 이벤트는 Outbox의 오버헤드 없이 직접 발행

**모든 이벤트에 같은 신뢰성 수준을 적용할 필요는 없다.** 이벤트의 중요도에 따라 전달 보장 수준을 달리하는 것이 설계다.

### 6. 버그 수정 및 안정성 개선

| 문제 | 수정 |
|------|------|
| OutboxRelayScheduler 멱등성 깨짐 | 릴레이 전 상태 체크 추가, 중복 발행 방지 |
| COUPON 토픽 누락 | 쿠폰 이벤트용 토픽 설정 추가 |
| 쿠폰 발급 안전성 | CouponIssueFacade에서 재고 소진/중복 발급 방어 로직 강화 |
| 미사용 이벤트 잔존 | ProductLikedEvent, ProductUnlikedEvent 삭제 (Kafka 전환으로 불필요) |
| 불필요한 DB 조회 | 이벤트 처리 시 불필요한 조회 제거 |
| EventLog status 하드코딩 | enum으로 타입 안전성 확보 |

### 7. 테스트

| 테스트 종류 | 내용 |
|------------|------|
| **단위 테스트** | CouponIssueRequest 상태 전이, EventEnvelope 직렬화, ProductMetrics 집계 로직 |
| **통합 테스트** | CouponIssueFacade 전체 흐름, OutboxEvent 저장 및 릴레이, CatalogEventProcessor DB 집계 |
| **E2E 테스트** | 선착순 쿠폰 발급 API → Kafka → 발급 완료 전 과정, 좋아요/주문 이벤트 Kafka 전달 |
| **DLQ 테스트** | 처리 실패 메시지가 DLQ로 정상 라우팅되는지 검증 |
| **Embedded Kafka** | EmbeddedKafkaTestSupport로 테스트 환경에서 Kafka 통합 테스트 기반 구축 |

---

## 이번 주의 깨달음

이번 주는 **"이벤트를 어디까지 분리할 것인가"**를 단계적으로 체감한 한 주였다.

처음에는 ApplicationEvent로 좋아요 집계를 분리하면서, 메인 트랜잭션에서 부가 로직을 빼는 것이 얼마나 코드를 깔끔하게 만드는지를 경험했다. 그런데 이벤트가 프로세스 내부에 머무르면, 그 프로세스가 죽으면 이벤트도 사라진다.

Transactional Outbox Pattern은 이 문제에 대한 답이었다. 이벤트를 DB에 먼저 저장하고, 별도 스케줄러가 Kafka로 릴레이하면 **트랜잭션의 원자성과 메시지 전달의 신뢰성을 동시에 확보**할 수 있다. 하지만 모든 이벤트에 이 수준의 신뢰성이 필요한 건 아니다. 조회 이벤트처럼 유실되어도 괜찮은 이벤트는 Outbox 없이 직접 발행하는 게 맞다.

선착순 쿠폰 발급에서는 **Redis로 빠르게 거르고, Kafka로 비동기 처리**하는 패턴을 적용했다. 동기로 처리하면 수천 건의 동시 요청이 DB를 직격하지만, Redis Lua로 원자적으로 검증하고 Kafka로 넘기면 DB는 Consumer의 속도로만 부하를 받는다.

Kafka 설정을 글로 정리하면서 잡은 이론이, 프로젝트에서 Producer/Consumer를 직접 설정할 때 그대로 적용됐다. `acks`, `retries`, DLQ 설정을 "왜 이 값이어야 하는가"를 알고 적용하니, 설정이 외울 대상이 아니라 **설계의 일부**가 됐다.

결국 이번 주에 배운 건 하나다 — **이벤트 기반 아키텍처는 "이벤트를 발행하는 것"이 아니라 "이벤트의 신뢰성 수준을 설계하는 것"**이다.
