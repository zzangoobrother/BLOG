# Cache Stampede - 캐시가 만료되는 순간 벌어지는 일

## 들어가며

캐시를 도입하면 성능이 좋아진다. 당연한 이야기다. 그런데 **캐시가 만료되는 바로 그 순간**, 오히려 캐시가 없을 때보다 더 심각한 장애가 발생할 수 있다. 이것이 **Cache Stampede(캐시 스탬피드)** 문제다.

"분명 캐시를 적용했는데, 왜 DB가 죽었지?"

이 질문의 답이 바로 Cache Stampede일 수 있다. 이 글에서는 Cache Stampede가 무엇이고, 왜 발생하며, 어떻게 해결하는지를 다룬다.

1. Cache Stampede란 무엇인가
2. 문제 시나리오 — 캐시 만료 시점에 벌어지는 일
3. 해결 전략 3가지 비교
4. 실전 적용 — Caffeine의 원자적 로딩

---

## Cache Stampede란 무엇인가

Cache Stampede(또는 Thundering Herd, Dog-pile Effect)는 **캐시가 만료되는 시점에 다수의 요청이 동시에 DB로 몰리는 현상**이다.

### 정상 상태

캐시가 유효할 때는 문제가 없다.

```
요청 1 → 캐시 Hit → 반환
요청 2 → 캐시 Hit → 반환
요청 3 → 캐시 Hit → 반환
           ↑
     캐시가 살아있으면
     DB에 요청이 가지 않는다
```

### Stampede 발생

캐시 TTL이 만료되는 순간, 상황이 급변한다.

```
시간   요청           캐시 상태        DB
─────────────────────────────────────────────
t0    (TTL 만료)      비어있음         대기 중
t1    요청 A          Miss → DB 조회   쿼리 실행 중...
t2    요청 B          Miss → DB 조회   쿼리 실행 중... (2개)
t3    요청 C          Miss → DB 조회   쿼리 실행 중... (3개)
t4    요청 D          Miss → DB 조회   쿼리 실행 중... (4개)
...
t5    요청 A 완료     캐시 저장        쿼리 완료
t6    요청 B 완료     캐시 덮어쓰기    쿼리 완료 (불필요했음)
t7    요청 C 완료     캐시 덮어쓰기    쿼리 완료 (불필요했음)
t8    요청 D 완료     캐시 덮어쓰기    쿼리 완료 (불필요했음)
```

**요청 A 하나만 DB를 조회하면 충분한데**, B, C, D 모두 같은 데이터를 중복 조회한다. 초당 1,000건의 요청이 들어오는 API라면, 캐시 만료 순간에 1,000건의 동일한 쿼리가 DB로 쏟아진다.

### 왜 위험한가

```
[일반적인 상황]
  요청 1,000건/초 → 캐시 Hit → DB 요청 0건

[TTL 만료 순간]
  요청 1,000건/초 → 캐시 Miss → DB 요청 1,000건 (동시)
                                  ↓
                          Connection Pool 고갈
                                  ↓
                          타임아웃 / 서비스 장애
```

특히 **무거운 쿼리**일수록 위험하다. 조인이 많거나 집계가 포함된 쿼리는 실행 시간이 길고, 그동안 더 많은 요청이 쌓인다. DB 커넥션 풀이 고갈되면 캐시와 무관한 다른 API까지 영향을 받는다.

---

## 문제 시나리오 — 멀티 레이어에서의 Stampede

단일 Redis 캐시에서도 Stampede가 발생하지만, **로컬 캐시를 추가했을 때**도 주의해야 한다.

### 로컬 캐시 → Redis 구간의 Stampede

```
[2계층 캐시에서의 Stampede]

Local Cache TTL 만료 시:
  스레드 A: Local Miss → Redis 조회
  스레드 B: Local Miss → Redis 조회 (동시)
  스레드 C: Local Miss → Redis 조회 (동시)
  스레드 D: Local Miss → Redis 조회 (동시)
```

Redis에서 데이터를 가져오는 것은 DB보다 빠르지만(~1ms), **동시에 수백 개의 스레드가 Redis를 조회하면 네트워크 대역폭과 Redis CPU에 부하**가 걸린다. Redis Hit이라 DB까지 가지 않더라도, 불필요한 중복 조회가 발생하는 것은 같다.

### Redis → DB 구간의 Stampede

```
Local Cache TTL 만료 + Redis Cache TTL 만료가 동시에 발생하면:
  스레드 A: Local Miss → Redis Miss → DB 조회
  스레드 B: Local Miss → Redis Miss → DB 조회 (동시)
  스레드 C: Local Miss → Redis Miss → DB 조회 (동시)
  ...
```

이 경우가 가장 위험하다. 모든 방어선이 동시에 무너지면서 DB에 직접 부하가 몰린다.

---

## 해결 전략 3가지 비교

### 전략 1: 뮤텍스(Mutex) 방식

캐시 미스 시 **하나의 스레드만 DB를 조회**하도록 락을 건다.

```
스레드 A: 캐시 Miss → 락 획득 → DB 조회 → 캐시 저장 → 락 해제
스레드 B: 캐시 Miss → 락 대기... → 락 획득 → 캐시 Hit → 반환
스레드 C: 캐시 Miss → 락 대기... → 락 획득 → 캐시 Hit → 반환
```

```java
public ProductInfo getProduct(Long productId) {
    ProductInfo cached = cache.get(productId);
    if (cached != null) {
        return cached;
    }

    Lock lock = lockMap.computeIfAbsent(productId, k -> new ReentrantLock());
    lock.lock();
    try {
        // 다른 스레드가 이미 캐시를 채웠을 수 있으므로 재확인
        cached = cache.get(productId);
        if (cached != null) {
            return cached;
        }

        ProductInfo product = db.findById(productId);
        cache.put(productId, product);
        return product;
    } finally {
        lock.unlock();
    }
}
```

| 장점 | 단점 |
|------|------|
| Stampede 완전 차단 | 직접 락 관리 필요 (락 맵, 메모리 누수 방지) |
| 구현 의도가 명확 | 분산 환경에서는 분산 락 필요 (Redis SETNX 등) |
| | 락 대기 시간만큼 응답 지연 |

### 전략 2: 스케줄러 기반 워밍

캐시가 만료되기 **전에** 주기적으로 갱신한다. TTL 만료 자체를 방지하는 전략이다.

```
워밍 주기: 2분
캐시 TTL:  3분

0m        2m        3m        4m
|--워밍----|--워밍----|--워밍----|
              ↑
     만료(3분) 전에 갱신 → Stampede 발생 여지 없음
```

```kotlin
@Scheduled(initialDelay = 0, fixedRate = 120_000)
fun warmProductListCache() {
    val topBrands = brandService.getTopBrands(limit = 10)

    for (brandId in topBrands.map { it.id } + listOf(null)) {
        for (sort in listOf("unsorted", "price:asc", "likes:desc")) {
            for (page in 0..2) {
                productCacheManager.getProducts(brandId, PageQuery(page, 20, sort))
            }
        }
    }
}
```

| 장점 | 단점 |
|------|------|
| Stampede 완전 차단 | 워밍 대상 키를 사전에 알아야 함 |
| 예측 가능한 부하 | 접근하지 않는 키도 워밍 (메모리 낭비) |
| 구현 단순 | **로컬 캐시에는 비효율적** (인스턴스마다 독립) |

**Redis 캐시(공유 캐시)**에는 효과적이다. 1대만 워밍하면 모든 인스턴스가 혜택을 받는다. 하지만 **로컬 캐시(인스턴스별 캐시)**에는 비효율적이다. 10대의 인스턴스가 있으면 10대 모두 워밍해야 한다.

### 전략 3: Caffeine 원자적 로딩 (Cache.get(key, loader))

Caffeine이 제공하는 `Cache.get(key, mappingFunction)`을 활용하는 방식이다.

```
스레드 A: Cache.get(key, loader)
  → 캐시 Miss → loader 실행 (이 스레드만)
  → Redis 조회 → 값 반환

스레드 B, C (같은 key에 동시 접근):
  → 스레드 A의 로딩 완료를 대기 → 같은 값 반환
```

```kotlin
// Caffeine이 내부적으로 키별 동기화를 처리한다
override fun getOrLoadProduct(
    productId: Long,
    loader: () -> ProductDetailInfo
): ProductDetailInfo {
    return detailCache.get(productId) { loader() }
    //                                ↑
    //                    같은 key에 대해 동시 호출되어도
    //                    loader는 단 한 번만 실행된다
}
```

내부적으로 어떻게 동작하는 걸까?

```
Caffeine의 Cache.get(key, loader) 내부 동작:

1. key에 대한 엔트리 확인
2. 엔트리 없음 → 해당 key에 대한 락 획득 (key별 세분화된 락)
3. 락 획득 성공 → loader 실행 → 값 저장 → 락 해제
4. 락 획득 실패 (다른 스레드가 이미 로딩 중) → 완료될 때까지 대기 → 같은 값 반환

핵심: "전체 캐시"가 아니라 "해당 key"에 대해서만 동기화
      → 다른 key의 조회는 전혀 영향받지 않음
```

| 장점 | 단점 |
|------|------|
| **Stampede 완전 차단** (키별 원자적 로딩) | 로딩 중 다른 스레드가 짧게 대기 |
| 접근하는 키만 캐싱 (lazy) | Caffeine 라이브러리에 의존 |
| 코드가 매우 단순 | |
| 키별 세분화된 락 → 다른 키 조회에 영향 없음 | |

---

## 세 가지 전략 비교

| 기준 | 뮤텍스 | 스케줄러 워밍 | Caffeine 원자적 로딩 |
|------|--------|-------------|---------------------|
| Stampede 방지 | O | O | **O** |
| 메모리 효율 | O (lazy) | X (전체 워밍) | **O (lazy)** |
| 구현 복잡도 | 높음 (락 관리) | 중간 | **낮음** |
| 분산 환경 | 분산 락 필요 | 1대만 실행 | 인스턴스별 독립 동작 |
| 적합한 캐시 계층 | Redis, DB | **Redis (공유 캐시)** | **로컬 캐시** |

결론적으로, **캐시 계층별로 다른 전략을 적용**하는 것이 가장 효과적이다.

```
[최적 조합]

로컬 캐시 (Caffeine)
  → Caffeine 원자적 로딩으로 Stampede 방지
  → lazy하게 접근하는 키만 캐싱
  → 인스턴스별 독립 동작, 별도 인프라 불필요

Redis 캐시 (공유 캐시)
  → 스케줄러 워밍으로 TTL 만료 전에 갱신
  → 1대만 워밍하면 모든 인스턴스가 혜택
  → 워밍 실패 시 Cache-Aside로 자연 복구
```

---

## 실전 적용 — 2계층 캐시에서의 Stampede 방지

실제 코드에서 두 전략이 어떻게 조합되는지 보자.

### 전체 흐름

```kotlin
class ProductCacheManager(
    private val productCacheRepository: ProductCacheRepository,
    private val productLocalCacheRepository: ProductLocalCacheRepository,
    private val productService: ProductService,
    private val brandService: BrandService,
) {
    fun getProduct(productId: Long): ProductDetailInfo {
        // 1단계: 로컬 캐시 조회 (Caffeine 원자적 로딩)
        //   → 같은 key에 동시 요청이 와도 loader는 한 번만 실행
        return productLocalCacheRepository.getOrLoadProduct(productId) {
            loadProductDetail(productId)
        }
    }

    private fun loadProductDetail(productId: Long): ProductDetailInfo {
        // 2단계: Redis 조회
        //   → 스케줄러 워밍 덕분에 대부분 Hit
        productCacheRepository.getProduct(productId)?.let { return it }

        // 3단계: Redis도 Miss → DB 조회
        val product = productService.getProduct(productId)
        val brand = brandService.getBrand(product.brandId)
        val detail = ProductDetailInfo.from(product, brand)

        // Redis에 저장 (다른 인스턴스를 위해)
        productCacheRepository.setProduct(productId, detail)
        return detail
    }
}
```

### 동시 요청 시나리오

```
[상품 ID=123에 대해 5개 스레드가 동시 요청]

스레드 A: getProduct(123)
  → Local Miss
  → Caffeine.get(123, loader) → loader 실행 시작
     → Redis 조회 → Hit → 값 반환
  → Local에 저장

스레드 B~E: getProduct(123)
  → Local Miss
  → Caffeine.get(123, loader) → 스레드 A의 로딩 대기 (~1ms)
  → 스레드 A의 결과를 함께 반환

결과:
  Redis 조회: 1회 (스레드 A만)
  DB 조회: 0회 (Redis Hit)
  모든 스레드: 같은 값을 반환
```

Caffeine의 원자적 로딩 덕분에, 5개 스레드가 동시에 접근해도 **Redis 조회는 단 1회**만 발생한다. 대기 시간도 Redis 조회 시간(~1ms) 수준이므로 실질적인 영향이 없다.

### Redis도 Miss인 최악의 경우

```
[서버 시작 직후, 워밍 전]

스레드 A: getProduct(123)
  → Local Miss
  → Caffeine.get(123, loader) → loader 실행
     → Redis Miss → DB 조회 → Redis 저장 → 값 반환
  → Local에 저장

스레드 B~E: getProduct(123)
  → Caffeine 대기 → 스레드 A의 결과를 반환

결과:
  DB 조회: 1회 (스레드 A만)
  Redis 저장: 1회
  5개 스레드 모두 정상 응답
```

Redis도 비어있는 최악의 상황에서도 DB 조회는 **1회**로 제한된다. 이것이 Stampede 방지의 핵심이다.

---

## 마무리

Cache Stampede는 캐시를 도입할 때 반드시 고려해야 하는 문제다. 캐시가 "있다가 없어지는 순간"이 가장 위험하다.

> **Cache Stampede란**
> - 캐시 TTL 만료 시 동일한 데이터에 대한 요청이 동시에 DB로 몰리는 현상
> - 초당 요청이 많을수록, 쿼리가 무거울수록 위험도가 급증

> **해결 전략**
> - 뮤텍스: 직접 락을 관리하여 하나의 스레드만 DB 조회 허용
> - 스케줄러 워밍: TTL 만료 전에 미리 갱신 → 공유 캐시(Redis)에 적합
> - Caffeine 원자적 로딩: 키별 자동 동기화 → 로컬 캐시에 적합

> **실전 조합**
> - 로컬 캐시: Caffeine `Cache.get(key, loader)` 원자적 로딩
> - Redis 캐시: 스케줄러 기반 워밍 (워밍 주기 < TTL)
> - 두 전략을 조합하면 어느 계층에서든 Stampede를 방지할 수 있다

캐시는 단순히 "데이터를 저장해두는 것"이 아니라, **만료, 갱신, 동시성**까지 설계해야 완성된다. 캐시를 도입할 때 "TTL이 만료되면 어떻게 되는가?"를 반드시 점검하자.

---

## 참고자료

- Martin Kleppmann, 「Designing Data-Intensive Applications」
- Ben Manes, [Caffeine Wiki - Population](https://github.com/ben-manes/caffeine/wiki/Population)
- Instagram Engineering, [Thundering Herds & Promises](https://instagram-engineering.com/thundering-herds-promises-82191c8af57d)
