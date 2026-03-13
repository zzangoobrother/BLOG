# 멀티 레이어 캐시 전략 - Local Cache와 Redis를 함께 쓰는 이유

## 들어가며

"Redis 캐시를 적용했는데, 더 빠르게 할 수 없을까?"

상품 조회 API에 Redis 캐시를 도입하면 DB 직접 조회 대비 응답 시간이 크게 줄어든다. 하지만 Redis도 결국 **네트워크를 타는 외부 저장소**다. 요청마다 네트워크 왕복(~1ms)이 발생하고, Redis 서버에 장애가 생기면 캐시가 통째로 무력화된다.

이 글에서는 Redis 위에 **로컬 캐시(Caffeine)**를 얹어 2계층 캐시 아키텍처를 구성한 경험을 공유한다. 다루는 내용은 다음과 같다.

1. 캐시 전략의 기본 - Cache-Aside 패턴
2. Redis 캐시만으로 부족한 이유
3. 2계층 캐시 아키텍처 설계
4. 캐시 워밍으로 Cold Start 방지
5. TTL 설계와 무효화 전략
6. DIP를 적용한 캐시 아키텍처

---

## 캐시 전략의 기본 - Cache-Aside 패턴

캐시 전략에는 여러 패턴이 있다. Cache-Aside, Write-Through, Write-Behind 등이 대표적이다. 이 중 **Cache-Aside(Look-Aside)**는 가장 널리 사용되는 패턴이다.

```
[Cache-Aside 패턴]

1. 캐시에서 데이터 조회
2. 캐시 Hit → 즉시 반환
3. 캐시 Miss → DB 조회 → 캐시에 저장 → 반환
```

```
요청 → 캐시 조회
         │
         ├─ Hit → 반환 (빠름)
         │
         └─ Miss → DB 조회
                     │
                     ├─ 캐시에 저장
                     └─ 반환
```

애플리케이션이 캐시를 직접 관리한다는 점이 특징이다. DB에 데이터를 쓸 때 캐시를 갱신하는 것이 아니라, **읽는 시점에 캐시가 없으면 DB에서 가져와 적재**한다. 구현이 단순하고 캐시 장애 시에도 DB로 폴백할 수 있어 안정적이다.

### 다른 캐시 패턴과의 비교

| 패턴 | 쓰기 시점 | 읽기 시점 | 특징 |
|------|----------|----------|------|
| **Cache-Aside** | 캐시에 쓰지 않음 | Miss 시 DB → 캐시 | 가장 범용적, 구현 단순 |
| Write-Through | DB + 캐시 동시 쓰기 | 항상 캐시 Hit | 쓰기 지연 발생, 일관성 높음 |
| Write-Behind | 캐시에만 쓰기 → 비동기 DB 반영 | 항상 캐시 Hit | 쓰기 빠름, 데이터 유실 위험 |

Cache-Aside는 **읽기 비중이 높은 워크로드**에 적합하다. 상품 조회처럼 쓰기보다 읽기가 압도적으로 많은 API에 잘 맞는다.

---

## Redis 캐시만으로 부족한 이유

Cache-Aside 패턴으로 Redis 캐시를 도입하면 구조는 이렇게 된다.

```
Controller → Facade → CacheManager → Redis → DB
```

Redis 캐시의 장점은 명확하다.

- DB 부하 감소 (캐시 Hit 시 DB 조회 없음)
- **공유 캐시** — 여러 애플리케이션 인스턴스가 같은 캐시를 사용
- TTL 기반 자동 만료

하지만 한계도 있다.

```
[Redis 캐시의 한계]

1. 네트워크 비용: 매 요청마다 Redis 서버와 TCP 왕복 (~1ms)
2. 단일 장애점: Redis 장애 시 모든 인스턴스에 영향
3. 직렬화 비용: 객체 → JSON → 객체 변환 오버헤드
```

특히 **초당 수만 건의 요청**이 들어오는 상품 목록 API에서는, 1ms의 네트워크 지연도 무시할 수 없다. 같은 데이터를 반복 조회하는데 매번 네트워크를 타야 할까?

---

## 2계층 캐시 아키텍처

해결책은 Redis 앞에 **로컬 캐시**를 하나 더 두는 것이다.

```
[Before] Controller → Facade → CacheManager → Redis → DB
[After]  Controller → Facade → CacheManager → Local(Caffeine) → Redis → DB
```

### 조회 흐름

```
사용자 요청
  │
  ▼
Local Cache 조회 (Caffeine, JVM 메모리)
  │
  ├─ Hit → 즉시 반환 (0ms, 네트워크 없음)
  │
  └─ Miss
       │
       ▼
     Redis 조회 (네트워크 ~1ms)
       │
       ├─ Hit → Local에 적재 후 반환
       │
       └─ Miss
            │
            ▼
          DB 조회 (~5-50ms)
            │
            ├─ Redis에 저장
            ├─ Local에 저장
            └─ 반환
```

### 왜 Caffeine인가

로컬 캐시 라이브러리로 Caffeine을 선택했다. Java/Kotlin 생태계에서 가장 성능이 뛰어난 인메모리 캐시다.

| 특징 | 설명 |
|------|------|
| 높은 적중률 | Window TinyLfu 기반 퇴출 정책 |
| 빠른 성능 | ConcurrentHashMap 기반, 락 경쟁 최소화 |
| 유연한 만료 정책 | 시간 기반, 크기 기반 만료 지원 |
| **원자적 로딩** | `Cache.get(key, loader)` — Cache Stampede 방지 |

### 구현 예시

Caffeine 캐시 저장소를 구성하면 다음과 같다.

```kotlin
class ProductLocalCacheRepositoryImpl : ProductLocalCacheRepository {

    // 상품 상세 정보 캐시
    private val detailCache: Cache<Long, ProductDetailInfo> = Caffeine.newBuilder()
        .maximumSize(1_000)           // 최대 1,000건
        .expireAfterWrite(60, TimeUnit.SECONDS)  // 60초 후 만료
        .build()

    // 상품 목록 캐시
    private val listCache: Cache<ProductListCacheKey, PageResult<ProductInfo>> = Caffeine.newBuilder()
        .maximumSize(200)             // 최대 200건
        .expireAfterWrite(5, TimeUnit.MINUTES)   // 5분 후 만료
        .build()

    override fun getOrLoadProduct(
        productId: Long,
        loader: () -> ProductDetailInfo
    ): ProductDetailInfo {
        return detailCache.get(productId) { loader() }
    }
}
```

핵심은 `detailCache.get(productId) { loader() }` 부분이다. Caffeine의 `Cache.get(key, loader)`는 **같은 키에 대해 동시에 여러 스레드가 접근해도 loader를 딱 한 번만 실행**한다. 이것이 Cache Stampede를 방지하는 핵심 메커니즘인데, 이 주제는 별도 글에서 깊이 다룰 예정이다.

### 2계층의 역할 분담

| 구분 | Local Cache (Caffeine) | Redis Cache |
|------|----------------------|-------------|
| **위치** | JVM 힙 메모리 | 외부 서버 |
| **접근 비용** | 0ms (메모리 직접 접근) | ~1ms (네트워크) |
| **공유 범위** | 해당 인스턴스만 | 모든 인스턴스 |
| **장애 영향** | 인스턴스 재시작 시 소멸 | Redis 장애 시 전체 영향 |
| **용도** | 핫 데이터 캐싱 | 공유 캐시, DB 부하 방어 |

로컬 캐시는 "자주 접근하는 핫 데이터"를 네트워크 없이 즉시 반환하는 역할이다. Redis는 여러 인스턴스가 공유하는 "2차 방어선"이다.

---

## 캐시 워밍으로 Cold Start 방지

2계층 캐시가 완성되었지만, 한 가지 문제가 남았다. **서버가 막 시작되었을 때 캐시가 비어있다.** 첫 번째 요청들은 모두 DB를 직접 조회해야 한다. 이를 **Cold Start 문제**라고 한다.

### 스케줄러 기반 워밍

서버 시작 직후, 그리고 주기적으로 인기 데이터를 미리 캐시에 적재한다.

```kotlin
@Component
class ProductCacheWarmingScheduler(
    private val productCacheManager: ProductCacheManager,
    private val brandService: BrandService,
) {
    @Scheduled(initialDelay = 0, fixedRate = 120_000) // 서버 시작 즉시 + 2분마다
    fun warmProductListCache() {
        val topBrands = brandService.getTopBrands(limit = 10)

        val brandIds = listOf(null) + topBrands.map { it.id }
        val sorts = listOf("unsorted", "price:asc", "likes:desc")

        for (brandId in brandIds) {
            for (sort in sorts) {
                for (page in 0..2) {
                    productCacheManager.getProducts(brandId, PageQuery(page, 20, sort))
                }
            }
        }
    }
}
```

### 워밍 대상

```
상위 10개 브랜드 + 전체(brandId=null)
  × 3가지 정렬 (최신순, 가격순, 좋아요순)
  × 3페이지 (0, 1, 2)
= 99건 캐시 항목
```

인기 있는 조회 조건을 사전에 캐싱하여, 대부분의 요청이 캐시에서 즉시 응답할 수 있도록 한다.

### 워밍 주기와 TTL의 관계

워밍이 효과를 발휘하려면 **워밍 주기가 TTL보다 짧아야** 한다.

```
Redis 목록 TTL: 3분
워밍 주기:      2분

[정상 상태] 워밍 주기 < TTL → 만료 전에 항상 갱신
0m        2m        3m        4m        5m
|--워밍----|--워밍----|--워밍----|--워밍----|
     ↑          ↑
  캐시 적재   만료(3m) 전에 갱신 → 끊김 없음

[워밍 실패 시] TTL 만료 후 Cache-Aside로 자연 복구
0m        2m        3m        4m
|--워밍----|--실패----|--만료----|--워밍----|
                      ↑         ↑
               캐시 만료    다음 워밍에서 복구
               (요청 시 Cache-Aside로 DB 조회)
```

워밍이 일시적으로 실패해도 Cache-Aside가 동작하므로 서비스에 영향이 없다. **스케줄러 워밍 + Cache-Aside의 조합이 안정적인 이유**다.

### Redis 워밍 vs Local 워밍

```
Redis 워밍:  1대만 실행 → 모든 인스턴스가 혜택 (공유 캐시)
Local 워밍:  각 인스턴스에서 실행 → 해당 인스턴스만 혜택

서버 시작 시:
  인스턴스 A의 스케줄러 실행
    → Redis 캐시 + 인스턴스 A의 로컬 캐시 워밍 완료

  인스턴스 B의 첫 요청
    → 로컬 캐시 Miss → Redis Hit (~1ms) → 로컬에 적재
    → 이후 요청부터 로컬 캐시 Hit (0ms)
```

---

## TTL 설계와 무효화 전략

### TTL 설계

캐시의 TTL은 **데이터 특성**에 따라 다르게 설정한다.

| 구분 | Local Cache (Caffeine) | Redis Cache |
|------|----------------------|-------------|
| 상품 상세 | 60초, 최대 1,000건 | 30초 |
| 상품 목록 | 5분, 최대 200건 | 3분 |

왜 로컬 캐시의 TTL이 Redis보다 긴가?

```
[Local TTL > Redis TTL인 이유]

Redis Miss → DB 조회 → Redis 저장 → Local 저장
                                      ↑
                               이 시점에 Local TTL 시작

Redis TTL 30초 만료 시, Local은 아직 유효할 수 있음
→ 로컬 캐시가 Redis Miss를 흡수하여 DB 부하를 추가 방어
```

다만 이는 **데이터 정합성과 트레이드오프**다. 로컬 캐시의 TTL이 길수록 오래된 데이터를 반환할 가능성이 높아진다. 상품 조회처럼 약간의 지연이 허용되는 데이터에 적합한 전략이다.

### 무효화 전략

TTL 기반 자동 만료 외에, 데이터 변경 시 **명시적 무효화**도 필요하다.

```kotlin
class ProductCacheManager(
    private val productCacheRepository: ProductCacheRepository,       // Redis
    private val productLocalCacheRepository: ProductLocalCacheRepository, // Local
) {
    fun evictProduct(productId: Long) {
        productLocalCacheRepository.evictProduct(productId)  // Local 캐시 제거
        productCacheRepository.evictProduct(productId)       // Redis 캐시 제거
    }

    fun evictAllProducts() {
        productLocalCacheRepository.evictAllProducts()       // Local 목록 캐시 전체 제거
        productCacheRepository.evictAllProducts()            // Redis 목록 캐시 전체 제거
    }
}
```

무효화 흐름은 **Local → Redis 순서**다. Local을 먼저 지우는 이유는, Redis를 먼저 지우면 그 사이에 들어온 요청이 Local의 오래된 데이터를 반환할 수 있기 때문이다.

### 캐시 키 설계

```kotlin
object ProductCachePolicy {
    // 상세 정보
    // key: "product:detail:123"
    fun detailKey(productId: Long) = "product:detail:$productId"

    // 목록 (다중 조건 포함)
    // key: "product:list:brand:5:price:asc:0:20"
    fun listKey(brandId: Long?, pageQuery: PageQuery): String {
        val brandPart = brandId?.let { "brand:$it:" } ?: ""
        return "product:list:${brandPart}${pageQuery.sort}:${pageQuery.page}:${pageQuery.size}"
    }

    // 목록 전체 삭제용 패턴
    // pattern: "product:list:*"
    fun listPattern() = "product:list:*"
}
```

목록 캐시 키에 **정렬 조건, 페이지, 사이즈**를 모두 포함한다. 같은 상품 목록이라도 정렬이 다르면 다른 캐시 키를 사용한다.

---

## DIP를 적용한 캐시 아키텍처

캐시 구현체(Caffeine, Redis)는 **인프라 기술**이다. 비즈니스 로직을 담당하는 Application 계층이 이런 기술에 직접 의존하면, 기술 교체 시 Application 코드까지 변경해야 한다.

### 인터페이스로 분리

```
Application 계층 (오케스트레이션)
  │
  ├── ProductCacheManager        ← 조회 순서 결정 (Local → Redis → DB)
  ├── ProductCacheRepository     ← 인터페이스 (Redis 캐시 추상화)
  └── ProductLocalCacheRepository ← 인터페이스 (로컬 캐시 추상화)

Infrastructure 계층 (구현)
  │
  ├── ProductCacheRepositoryImpl      ← Redis 구현체
  └── ProductLocalCacheRepositoryImpl ← Caffeine 구현체
```

`ProductCacheManager`는 "어떤 순서로 캐시를 조회하고, 미스 시 어디서 데이터를 가져올 것인가"라는 **오케스트레이션만** 담당한다.

```kotlin
// Application 계층 - 인터페이스
interface ProductLocalCacheRepository {
    fun getOrLoadProduct(productId: Long, loader: () -> ProductDetailInfo): ProductDetailInfo
    fun getOrLoadProducts(brandId: Long?, pageQuery: PageQuery, loader: () -> PageResult<ProductInfo>): PageResult<ProductInfo>
    fun evictProduct(productId: Long)
    fun evictAllProducts()
    fun evictAll()
}

// Application 계층 - 오케스트레이션
class ProductCacheManager(
    private val productCacheRepository: ProductCacheRepository,
    private val productLocalCacheRepository: ProductLocalCacheRepository,
    private val productService: ProductService,
    private val brandService: BrandService,
) {
    fun getProduct(productId: Long): ProductDetailInfo {
        return productLocalCacheRepository.getOrLoadProduct(productId) {
            loadProductDetail(productId)  // Local Miss 시 Redis → DB 순서로 조회
        }
    }

    private fun loadProductDetail(productId: Long): ProductDetailInfo {
        // Redis에서 조회
        productCacheRepository.getProduct(productId)?.let { return it }

        // Redis도 Miss → DB 조회
        val product = productService.getProduct(productId)
        val brand = brandService.getBrand(product.brandId)
        val detail = ProductDetailInfo.from(product, brand)

        // Redis에 저장
        productCacheRepository.setProduct(productId, detail)
        return detail
    }
}
```

### 이 구조의 장점

```
[의존성 방향]

ProductCacheManager (application)
    │
    ├──→ ProductLocalCacheRepository (application, 인터페이스)
    │         ▲
    │         │ 구현
    │    ProductLocalCacheRepositoryImpl (infrastructure, Caffeine)
    │
    └──→ ProductCacheRepository (application, 인터페이스)
              ▲
              │ 구현
         ProductCacheRepositoryImpl (infrastructure, Redis)
```

- Application 계층은 Caffeine이나 Redis를 알지 못한다
- 캐시 기술을 교체해도 인터페이스 구현체만 바꾸면 된다
- 테스트 시 캐시를 Mock으로 대체할 수 있다

---

## 마무리

2계층 캐시 아키텍처를 정리하면 다음과 같다.

> **왜 2계층인가**
> - Local Cache: 네트워크 없이 0ms 응답, 핫 데이터 즉시 반환
> - Redis Cache: 인스턴스 간 공유, DB 부하 방어의 2차 방어선
> - DB: 최종 데이터 소스

> **캐시 전략**
> - Cache-Aside 패턴: 읽기 시점에 캐시 적재, 구현 단순, 장애 시 DB 폴백
> - 스케줄러 워밍: Cold Start 방지, 인기 데이터 99건 사전 적재
> - 명시적 무효화: 데이터 변경 시 Local → Redis 순서로 제거

> **아키텍처 설계**
> - DIP 적용: 캐시 기술(Caffeine, Redis)을 인터페이스 뒤로 숨김
> - CacheManager: 조회 순서 오케스트레이션만 담당
> - 각 캐시 구현체: Infrastructure 계층에 배치

캐시는 도입하는 것보다 **잘 관리하는 것**이 어렵다. TTL 설계, 무효화 타이밍, 데이터 정합성 사이의 트레이드오프를 이해하고, 서비스 특성에 맞는 전략을 선택하는 것이 핵심이다.

---

## 참고자료

- Martin Kleppmann, 「Designing Data-Intensive Applications」
- Ben Manes, [Caffeine - A high performance caching library for Java](https://github.com/ben-manes/caffeine)
- Redis Documentation, [Client-side caching](https://redis.io/docs/manual/client-side-caching/)
