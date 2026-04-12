# WIL - Redis ZSET 적재에서 Reconciliation까지 — 랭킹 시스템을 관통한 한 주

## 이번 주 요약

이번 주는 **"오늘의 인기 상품" 랭킹 시스템을 끝에서 끝까지 설계하고 구현**한 한 주였다. 대기열이 지난 주의 주제였다면, 이번 주는 **랭킹**이다. Redis ZSET에 점수를 적재하는 Streamer 파이프라인을 먼저 올리고, 일별 키로 시간을 잘라 과거의 영향력을 끊었다. Carry-Over로 콜드 스타트를 완화하면서 양자화가 어떻게 감쇠를 흉내 내는지를 글과 코드 양쪽에서 관통했다. 조회 API와 상품 상세의 순위 노출을 붙이고, 조회/적재 양쪽에 Redis 장애 대응을 얹었다. 마지막으로 Redis가 유실되더라도 복구할 수 있도록, DB 집계값으로 Redis 점수를 다시 맞추는 Reconciliation 배치까지 구현했다. 발행 글에서는 **"거부와 대기는 설계 철학이 다르다"**로 대기열 시리즈를 마무리하고, **"과거는 현재보다 덜 중요해야 한다"**로 랭킹의 시간 설계 본질을 정리했다.

---

## 이번 주에 정리한 것

### 거부와 대기는 설계 철학이 다르다

Rate Limiting과 Queuing은 같은 과부하를 다른 관점으로 본다.

- **Rate Limiting**: 초과 트래픽을 **적**으로 본다. 429로 잘라낸다. 비용이 거의 0. 대신 유저의 재시도 폭풍이 원래 문제보다 더 큰 부하를 만든다.
- **Queuing**: 초과 트래픽을 **고객**으로 본다. 순번을 부여한다. 인프라 비용이 들고 이탈 관리라는 새 문제가 생긴다. 대신 재시도 폭풍이 구조적으로 사라진다.
- **실전은 조합**: 봇은 Rate Limiting으로 걸러내고, 정상 유저만 대기열에 진입시킨다. 초과 트래픽 중에서 **적과 고객을 분리**하는 것이 핵심이다.

판단 기준은 하나로 줄어든다 — **"이 요청의 주인이 기다려서라도 결과를 얻고 싶은 사람인가?"**

---

### 과거는 현재보다 덜 중요해야 한다

하나의 ZSET에 점수를 누적하면, 6개월 전 상품이 오늘의 신상품을 영원히 지배한다. "오늘의 인기"를 보여주겠다고 해놓고 **역대 누적**을 보여주는 모순. 과거의 영향력을 줄이는 길은 두 가지다.

| | 자르기 (양자화) | 녹이기 (감쇠) |
|---|---|---|
| 방식 | 일별 키 분리, 매일 0에서 시작 | score × decay(time) |
| 장점 | 설명/디버깅/운영이 쉬움 | 콜드 스타트 없음 |
| 단점 | 경계에서 단절(콜드 스타트) | 조회 시점마다 결과 달라짐 |
| 보완 | Carry-Over (전날 점수 × 0.1) | — |

핵심 통찰은 **Carry-Over가 결국 감쇠를 흉내 낸다**는 것이다. 하루 지나면 10%, 이틀 지나면 1%. 이산적 계단식 감쇠. 시간을 자른 다음 경계를 부드럽게 잇는 행위가 결국 감쇠와 같은 효과를 낸다.

그리고 어떤 방식을 골라도 **Matthew Effect(노출이 노출을 만드는 자기 강화)**는 스케일만 바뀔 뿐 사라지지 않는다. 시간 설계만으로는 해결할 수 없는 별도의 문제라는 점을 인정하는 것도 설계의 일부다.

---

## 이번 주에 구현한 것

### 1. Redis ZSET 기반 랭킹 저장소와 적재 파이프라인

Kafka에서 받은 상품 이벤트와 주문 이벤트를 Redis ZSET에 점수로 적재했다.

| 구성 요소 | 역할 |
|----------|------|
| `RankingService` (streamer) | 이벤트 타입별 가중치 규칙 캡슐화 (VIEW 0.1, LIKE 0.2, ORDER 0.7×log10(금액)) |
| `RankingRedisRepository` (streamer) | `ZINCRBY` + 조건부 TTL. Lua 스크립트로 원자적 실행 |
| `CatalogEventProcessor` | VIEW/LIKE/UNLIKE 이벤트를 받아 점수 반영 |
| `OrderEventProcessor` | ORDER_COMPLETED 이벤트에서 주문 금액 기반 점수 반영 |
| `ranking_increment.lua` | `ZINCRBY` → `TTL == -1`일 때만 `EXPIRE` 설정. 두 명령을 원자적으로 묶음 |

**일별 키 전략**: `ranking:all:YYYYMMDD`. 시간을 하루 단위로 양자화한다. TTL은 2일 — Carry-Over가 내일 키를 만든 뒤에도 전날 키가 살아있도록 하루의 여유를 둔다.

**주문 점수의 log 스케일링**:
```kotlin
val score = ORDER_WEIGHT * log10(totalAmount.toDouble())
```
100만원 주문이 1만원 주문의 100배 점수를 받으면, 고가 상품이 랭킹을 독식한다. log10으로 스케일을 압축해서 **"많이 팔린 것"과 "비싸게 팔린 것"의 균형**을 맞췄다.

**TTL 설정을 Lua로 묶은 이유**: `ZINCRBY` 후 별도로 `EXPIRE`를 호출하면, 첫 호출과 두 번째 호출 사이에 키가 조회되거나 복제될 수 있다. `TTL == -1` 체크를 Lua 내부로 옮겨서 **재호출 때마다 TTL을 덮어쓰지 않도록** 막는 동시에 원자성도 확보했다.

### 2. 랭킹 조회 API와 상품 상세의 순위 노출

랭킹 조회는 읽기 중심이므로 commerce-api에 분리해서 배치했다.

| API | 설명 |
|-----|------|
| `GET /api/v1/rankings?date=20260412&page=1&size=20` | 일별 Top N 조회. `ZREVRANGE` + `WITHSCORES` 사용 |
| `GET /api/v1/products/{id}` | 상품 상세에 당일 순위 포함. `ZRANK` O(log N) |

**페이지네이션 검증**: `page >= 1`, `size는 1~100`. 정렬/필터를 DB에서 했다면 ORDER BY 비용이 조회마다 발생하지만, ZSET은 정렬된 상태로 저장되므로 조회는 O(log N + M)이다.

**상품 상세의 rank 0-based → 1-based 변환**: Redis `ZRANK`는 0부터 시작한다. 유저에게는 "1위"로 보여줘야 하므로 `zeroBasedRank + 1`. 작은 세부사항이지만 이런 off-by-one이 결국 "1등 없는 랭킹"을 만든다.

### 3. Carry-Over 스케줄러

콜드 스타트를 완화하기 위해 매일 23:50에 전날 점수 × 0.1을 다음 날 키에 복사했다.

| 구성 요소 | 설명 |
|----------|------|
| `RankingCarryOverScheduler` | `@Scheduled(cron = "0 50 23 * * *")`. `@Profile("!test")`로 테스트 환경 제외 |
| `ranking_carry_over.lua` | `DEL toKey` → `ZUNIONSTORE toKey 1 fromKey WEIGHTS 0.1` → `EXPIRE` |

```lua
redis.call('DEL', KEYS[2])
redis.call('ZUNIONSTORE', KEYS[2], 1, KEYS[1], 'WEIGHTS', ARGV[1])
redis.call('EXPIRE', KEYS[2], tonumber(ARGV[2]))
```

가중치 0.1의 의미는 글에서 정리한 그대로다. 100%면 누적과 동일해서 양자화의 의미가 사라지고, 0%면 콜드 스타트가 그대로 남는다. **10%는 "기억하되 지배당하지 않는" 균형점**이다. 이틀만 지나면 전전날 점수는 1%(0.1 × 0.1)로 사실상 소멸한다.

23:50에 실행하는 이유는 자정 직전에 내일 키를 미리 만들어두기 위해서다. 자정에 새 요청이 오면 이미 전날 점수의 10%가 반영된 키에 쌓인다.

### 4. Redis 장애 대응 — 조회와 적재의 분리

랭킹은 **보조 기능**이다. 랭킹이 안 보인다고 주문/조회가 막히면 본말전도다. 이 원칙을 조회와 적재 양쪽에 다르게 적용했다.

**조회 (commerce-api): Redis 실패 시 DB fallback**

```kotlin
val entries = runCatching {
    rankingService.getTopRankings(date, page, size)
}.getOrElse {
    log.warn("[RankingFacade] Redis 랭킹 조회 실패, DB fallback 수행", it)
    rankingService.getTopRankingsFromDb(page, size)
}
```

DB fallback은 `product_metrics` 테이블의 `view_count`, `like_count`, `sales_count`에 같은 가중치를 곱해 ORDER BY로 계산한다. 정확도는 떨어지지만(log 스케일링 없음, Carry-Over 반영 없음) **"보이는 랭킹"**을 유지한다. 비상시 근사값이라도 보여주는 것이 목적이다.

**적재 (commerce-streamer): Redis 실패해도 Kafka 처리 성공**

```kotlin
productStockRepository.decrementStock(item.productId, item.quantity)
productMetricsRepository.incrementSalesCount(item.productId, item.quantity)
runCatching {
    rankingService.updateScoreForOrder(today, item.productId, item.unitPrice, item.quantity)
}.onFailure {
    log.warn("[Order] 랭킹 점수 업데이트 실패: productId={}", item.productId, it)
}
```

재고 차감과 메트릭 업데이트(DB)는 반드시 성공해야 한다. 랭킹 업데이트(Redis)는 실패해도 괜찮다. `runCatching`으로 묶어서 **Redis 장애가 Kafka 재처리 루프를 만들지 않도록** 차단했다. 상품 상세 API의 순위 조회에도 같은 패턴을 적용했다 — 순위를 못 가져온다고 상품 상세 자체가 실패하면 안 된다.

### 5. DB 기반 Reconciliation 배치

Redis 장애로 일부 이벤트가 유실되거나, Carry-Over 가중치 계산이 흐트러지면 Redis와 DB 집계값이 벌어진다. 하루 단위로 이를 맞추는 배치를 구현했다.

| 구성 요소 | 역할 |
|----------|------|
| `RankingReconciliationJobConfig` | Spring Batch Job. chunk size 100 |
| `RankingReconciliationReader` | `JdbcPagingItemReader`로 `product_metrics` 페이징 조회 |
| `RankingReconciliationProcessor` | DB 점수 vs Redis 점수 비교. **차이가 0.001 이상일 때만** 통과 |
| `RankingReconciliationWriter` | 불일치 항목만 `ZADD`로 덮어쓰기. TTL 누락 시 설정 |

```kotlin
// Processor
val redisScore = masterRedisTemplate.opsForZSet().score(key, item.productId.toString())
val dbScore = item.calculateScore()

if (redisScore != null && kotlin.math.abs(redisScore - dbScore) < 0.001) {
    return null   // 일치 → 스킵
}
return item        // 불일치 → Writer로
```

**Processor에서 걸러내는 이유**: 불일치가 없는 상품도 전부 `ZADD`하면 Redis에 불필요한 쓰기가 발생한다. 쇼핑몰 규모가 커지면 `product_metrics`가 수만~수십만 건이다. 그 중 실제 불일치는 수십~수백 건일 것이다. **Processor에서 null을 반환하면 Writer에 전달되지 않는 Spring Batch의 필터 규칙**을 활용해 필요한 건만 통과시켰다.

**0.001 허용 오차**: double 연산의 부동소수점 오차와 log10 계산의 반올림을 감안했다. 완전 일치를 요구하면 매일 모든 키가 "불일치"로 보고된다.

이 배치가 있으면 Redis가 통째로 날아가도 DB의 `product_metrics`만 살아있으면 랭킹을 재구축할 수 있다. **"Redis는 캐시, DB가 진실의 원천"**이라는 전제를 Reconciliation이 보장한다.

### 6. 코드 리뷰 반영 — Critical 3건 + Important 2건

혼자 구현한 뒤 코드 리뷰를 돌렸다. 주요 수정:

| 종류 | 수정 내용 |
|------|----------|
| Critical | 날짜 파라미터 검증 미흡 → `yyyyMMdd` 파싱 실패 시 400 응답 |
| Critical | `ranking_carry_over.lua`에서 `DEL` 누락 시 기존 데이터와 병합되는 이슈 |
| Critical | 페이지네이션 경계값 검증 (page < 1, size > 100) |
| Important | `RankingController` 메서드 추출 (`parseDate`, `validatePagination`) |
| Important | 스케줄러 레이어의 `Profile("!test")` 명시로 E2E 간섭 차단 |

리뷰를 통해 알게 된 것 — **Lua 스크립트는 한 번 실행되면 다시 돌릴 때 기존 키 상태를 의식해야 한다.** `ZUNIONSTORE`는 destination에 이미 데이터가 있으면 union 결과를 덮어쓰지 않고 합친다. `DEL`을 먼저 호출해야 의도대로 동작한다.

### 7. 테스트

| 테스트 종류 | 내용 |
|------------|------|
| **단위 테스트** | `RankingService` 가중치 계산, `RankingScore.calculateScore()`, 음수 금액 스킵 |
| **통합 테스트** | `RankingRedisRepositoryTest` Redis ZSET 조회/적재, Carry-Over Lua, TTL 유지 검증 |
| **동시성 테스트** | Kafka 재전송으로 인한 동일 이벤트 중복 처리 방지 (멱등성 테이블 + `insertIgnore`) |
| **E2E 테스트** | `RankingApiE2ETest` — 이벤트 수신부터 ZSET 적재, 조회 API 응답까지 |
| **Cross-stream 테스트** | `CrossStreamVersionIntegrationTest` — Catalog/Order 이벤트가 순서 없이 도착해도 최신성 보장 |
| **HTTP 요청 파일** | `ranking-v1.http` 추가 |

특히 E2E 테스트에서 신경 쓴 부분은 **날짜 격리**였다. 테스트가 오늘 키를 오염시키면 다음 테스트가 영향을 받는다. 각 테스트마다 고유 날짜를 쓰거나 종료 시 `DEL`로 정리했다.

---

## 이번 주의 깨달음

이번 주는 **"비용은 사라지지 않는다, 옮길 뿐이다"**를 구현 내내 체감한 한 주였다.

DB ORDER BY로 랭킹을 만들면 조회마다 정렬 비용을 치른다. Redis ZSET은 `ZINCRBY` 시점에 O(log N)을 치르고, 조회는 저장된 정렬 순서를 그대로 읽기만 한다. **"성능 개선"이라고 부르지만 실은 비용을 읽기에서 쓰기로 옮긴 것**이다. 읽기:쓰기 비율이 99:1 수준인 랭킹에서는 이 이동이 명백한 이득이지만, 로그 적재처럼 쓰기가 압도적인 시스템에서는 반대가 맞다.

같은 패턴이 반복된다. 인덱스는 조회 비용을 INSERT/UPDATE로 옮긴다. 캐시는 읽기 비용을 캐시 적재 시점으로 옮긴다. CQRS는 읽기/쓰기 모델을 분리해서 각각 최적화한다. **"어디서 비용을 치를 것인가"가 설계의 본질**이라는 걸, 랭킹을 구현하면서 손으로 확인했다.

두 번째 깨달음은 **"시간을 다루는 방식이 설계의 성격을 정한다"**는 것이다.

하나의 키에 누적하면 과거가 현재를 지배한다. 일별 키로 자르면 오늘의 실적으로 깨끗하게 경쟁한다. 하지만 자르면 경계에서 단절이 생기고, 새벽 0시의 랭킹은 비어있다. Carry-Over 10%로 메웠더니 결국 **양자화가 감쇠를 흉내 내는** 형태가 됐다. 자르기와 녹이기는 다른 설계 같지만, "과거는 현재보다 덜 중요해야 한다"는 같은 전제에서 출발한다. 구현이 달라도 **철학은 같다.**

세 번째 깨달음은 **"장애 대응은 층위별로 설계해야 한다"**는 것이다.

Redis가 죽으면 랭킹이 전면 실패해야 하는가? 그럴 수 없다. 조회에는 DB fallback이 있고, 적재에는 runCatching이 있다. 두 층위의 목적이 다르다. 조회 fallback은 **"보이는 랭킹을 유지"**하기 위함이고, 적재 runCatching은 **"Redis 장애가 본연의 비즈니스(재고/주문)를 막지 않게"** 하기 위함이다. 여기에 Reconciliation 배치가 얹어지면 **"Redis가 완전히 유실돼도 DB로 재구축 가능"**이라는 복구 지점이 생긴다. 지난 주 CircuitBreaker + Bulkhead가 실시간 장애 대응이었다면, 이번 주 Reconciliation은 **사후 복구**의 설계다.

돌아보면 지난 주와 이번 주는 같은 맥락이다. 지난 주는 "속도를 조절하는 시스템"을, 이번 주는 "과거의 무게를 조절하는 시스템"을 만들었다. 대기열은 트래픽의 시간을 조절하고, 랭킹의 일별 키는 데이터의 시간을 조절한다. 양쪽 다 **"문제를 없애는 것이 아니라, 시스템이 감당할 수 있는 크기로 줄이는 일"**이었다.

다음 주는 이 랭킹 시스템 위에 **개인화/추천**이 올라갈 차례다. Matthew Effect가 본격적으로 문제가 되는 지점이고, Exploration/Exploitation 같은 또 다른 차원의 설계가 필요해진다. 시간을 다루는 것만으로는 해결되지 않는 문제를 어떻게 다룰 것인가 — 이번 주에 남겨둔 숙제다.
