# 동시성과 Lock의 개념 - DB와 JPA편

## 들어가며

[이전 글](https://zzangoobrother.github.io/BLOG/?post=%5B20260304%5D_%5B%EB%8F%99%EC%8B%9C%EC%84%B1%EA%B3%BC+Lock%EC%9D%98+%EA%B0%9C%EB%85%90+-+%EC%88%9C%EC%88%98+Java%ED%8E%B8%5D_%5B%EB%8F%99%EC%8B%9C%EC%84%B1%5D_%5B%5D_%5B%EB%8F%99%EC%8B%9C%EC%84%B1%EA%B3%BC+Lock+%EA%B0%9C%EB%85%90%2C+%EC%88%9C%EC%88%98+Java%EC%97%90%EC%84%9C%EC%9D%98+%EB%8F%99%EC%8B%9C%EC%84%B1+%EC%A0%9C%EC%96%B4%2C+CAS+%EC%9B%90%EB%A6%AC%5D_%5B%5D.md)에서는 **순수 Java 관점**에서 동시성을 다뤘다. `synchronized`, `ReentrantLock`, `CAS`까지 — 애플리케이션 레벨의 동시성 제어 도구들이었다.

하지만 현실의 백엔드 서비스에서는 **데이터베이스**가 끼어든다. 아무리 Java 코드에서 동기화를 잘 해도, DB에서 데이터 정합성이 깨지면 의미가 없다. 특히 JPA를 사용하면 **1차 캐시**라는 변수까지 추가된다.

이번 글에서 다루는 내용은 다음과 같다.

1. DB 격리 수준과 세 가지 읽기 이상 현상
2. JPA 1차 캐시가 만드는 데이터 정합성 함정
3. 낙관적 락과 비관적 락 — 실전 코드와 함께

---

## DB 격리 수준(Isolation Level)

### 트랜잭션과 동시성의 딜레마

트랜잭션은 **ACID** 중 Isolation(격리성)을 보장해야 한다. 이상적으로는 모든 트랜잭션이 서로 완전히 격리되어야 하지만, 그렇게 하면 **동시 처리 성능이 바닥**을 친다. 모든 트랜잭션을 순차적으로 실행하는 것과 다를 바 없기 때문이다.

그래서 SQL 표준은 **네 가지 격리 수준**을 정의했다. 격리 수준이 낮을수록 동시 처리 성능은 올라가지만, 데이터 정합성 문제가 발생할 수 있다.

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
|-----------|:----------:|:-------------------:|:------------:|
| READ UNCOMMITTED | O | O | O |
| READ COMMITTED | X | O | O |
| REPEATABLE READ | X | X | O |
| SERIALIZABLE | X | X | X |

이 세 가지 읽기 이상 현상을 하나씩 살펴보자.

---

### Dirty Read — 커밋되지 않은 데이터 읽기

**Dirty Read**는 다른 트랜잭션이 아직 **커밋하지 않은** 데이터를 읽는 현상이다.

```
시간  트랜잭션A                     트랜잭션B                    products.stock
─────────────────────────────────────────────────────────────────────────────
t1   BEGIN                                                      100
t2   UPDATE stock = 50                                          50 (미커밋)
t3                                 BEGIN
t4                                 SELECT stock → 50 (!)        50 (미커밋)
t5   ROLLBACK                                                   100
t6                                 -- 50을 기반으로 비즈니스 로직 수행
t7                                 COMMIT
```

트랜잭션B는 트랜잭션A가 **롤백할 데이터**를 읽어버렸다. 존재하지 않았던 값을 기반으로 의사결정을 한 셈이다. 재고가 100인데 50으로 읽어서 "재고 부족"이라고 판단할 수 있다.

**READ UNCOMMITTED**에서만 발생하며, 실무에서 이 격리 수준을 쓰는 경우는 거의 없다.

> MySQL의 기본 격리 수준은 **REPEATABLE READ**, PostgreSQL은 **READ COMMITTED**이다. 둘 다 Dirty Read는 발생하지 않는다.

---

### Non-Repeatable Read — 같은 쿼리, 다른 결과

**Non-Repeatable Read**는 같은 트랜잭션 안에서 **같은 행을 두 번 읽었을 때 값이 달라지는** 현상이다.

```
시간  트랜잭션A                     트랜잭션B                    products.stock
─────────────────────────────────────────────────────────────────────────────
t1   BEGIN
t2   SELECT stock → 100                                        100
t3                                 BEGIN
t4                                 UPDATE stock = 50
t5                                 COMMIT                      50
t6   SELECT stock → 50 (!)                                     50
t7   COMMIT
```

트랜잭션A가 같은 상품의 재고를 두 번 조회했는데, 첫 번째는 100, 두 번째는 50이다. 그 사이에 트랜잭션B가 값을 변경하고 커밋했기 때문이다.

이것이 왜 문제일까? 재고를 확인하고 → 주문을 생성하는 로직을 생각해보자.

```java
@Transactional
public void placeOrder(Long productId, int quantity) {
    // 1. 재고 확인: 100개
    Product product = productRepository.findById(productId);

    // 2. 여기서 다른 트랜잭션이 재고를 50으로 변경!

    // 3. 재고 차감 로직 — 100개 기준으로 계산
    product.deductStock(quantity); // 의도와 다른 결과
}
```

**READ COMMITTED**에서 발생하며, **REPEATABLE READ** 이상에서는 방지된다. MySQL(InnoDB)은 **MVCC(Multi-Version Concurrency Control)**를 사용하여 트랜잭션 시작 시점의 스냅샷을 보여주므로, 같은 트랜잭션 안에서 같은 쿼리는 항상 같은 결과를 반환한다.

---

### Phantom Read — 없던 행이 나타나다

**Phantom Read**는 같은 조건으로 조회했을 때 **행의 개수가 달라지는** 현상이다. Non-Repeatable Read가 "값의 변경"이라면, Phantom Read는 "행의 추가/삭제"다.

```
시간  트랜잭션A                              트랜잭션B
──────────────────────────────────────────────────────────────
t1   BEGIN
t2   SELECT * FROM orders
     WHERE user_id = 1 → 3건
t3                                          BEGIN
t4                                          INSERT INTO orders
                                            (user_id = 1, ...)
t5                                          COMMIT
t6   SELECT * FROM orders
     WHERE user_id = 1 → 4건 (!)
t7   COMMIT
```

트랜잭션A가 같은 조건으로 두 번 조회했는데, 두 번째에 새로운 행이 "유령(Phantom)"처럼 나타났다. 이 현상은 **REPEATABLE READ**에서도 이론적으로는 발생할 수 있다.

단, MySQL(InnoDB)은 **Gap Lock**과 **Next-Key Lock**을 사용하여 REPEATABLE READ에서도 Phantom Read를 대부분 방지한다. 이 부분은 다음 편(DB Lock편)에서 자세히 다룰 예정이다.

---

### 세 가지 현상 비교

| 현상 | 무엇이 변하는가 | 원인 |
|------|----------------|------|
| Dirty Read | **값** (미커밋) | 다른 트랜잭션의 커밋되지 않은 변경을 읽음 |
| Non-Repeatable Read | **값** (커밋됨) | 다른 트랜잭션이 커밋한 UPDATE를 읽음 |
| Phantom Read | **행의 개수** | 다른 트랜잭션이 커밋한 INSERT/DELETE 반영 |

격리 수준을 선택하는 기준은 명확하다. **데이터 정합성이 중요하면 격리 수준을 올리고, 동시 처리 성능이 중요하면 낮춘다.** 대부분의 OLTP 서비스에서는 **READ COMMITTED** 또는 **REPEATABLE READ**를 사용한다.

---

## JPA 1차 캐시와 데이터 정합성

### 1차 캐시란?

JPA의 `EntityManager`는 **1차 캐시(First-Level Cache)**를 내장하고 있다. 영속성 컨텍스트(Persistence Context)라고도 부른다. 같은 트랜잭션 안에서 같은 엔티티를 여러 번 조회하면, 두 번째부터는 DB에 쿼리를 날리지 않고 **캐시에서 반환**한다.

```java
@Transactional
public void example() {
    // 1. DB에서 조회 → 1차 캐시에 저장
    Product product1 = productRepository.findById(1L);

    // 2. 1차 캐시에서 반환 (DB 쿼리 X)
    Product product2 = productRepository.findById(1L);

    // 같은 객체
    assert product1 == product2; // true
}
```

이 설계는 성능과 **동일성 보장(Identity Guarantee)**이라는 두 가지 이점을 제공한다. 하지만 동시성 환경에서는 **함정**이 된다.

---

### 함정 1: 1차 캐시가 DB 변경을 가리는 경우

```java
@Transactional
public void riskyOperation(Long productId) {
    // 1. 상품 조회 → 1차 캐시에 저장 (stock = 100)
    Product product = productRepository.findById(productId);

    // 2. 여기서 다른 스레드가 DB에서 직접 stock을 50으로 변경!

    // 3. 다시 조회해도 1차 캐시에서 반환 (stock = 100)
    Product sameProduct = productRepository.findById(productId);
    // sameProduct.getStockQuantity() → 여전히 100

    // 4. DB의 실제 값(50)과 다른 값(100)을 기반으로 로직 수행
}
```

1차 캐시는 **트랜잭션 시작 시점의 상태를 유지**한다. DB에서 값이 변경되어도 캐시가 갱신되지 않으므로, 오래된 데이터를 기반으로 비즈니스 로직을 수행하게 된다.

이 문제가 실제로 발생하는 대표적인 시나리오가 있다.

---

### 함정 2: Native Query와 1차 캐시의 불일치

실제 커머스 프로젝트에서 좋아요 기능을 구현할 때 이 문제를 마주할 수 있다.

```kotlin
// ProductJpaRepository.kt
@Modifying(clearAutomatically = true)
@Query("UPDATE products SET likes = likes + 1 WHERE id = :productId", nativeQuery = true)
fun incrementLikeCount(productId: Long)
```

이 코드에서 `clearAutomatically = true`가 없다면 어떻게 될까?

```java
@Transactional
public void likeAndCheck(Long productId) {
    // 1. 상품 조회 → 1차 캐시에 저장 (likes = 10)
    Product product = productRepository.findById(productId);

    // 2. Native Query로 DB 직접 업데이트 (likes = 11)
    productRepository.incrementLikeCount(productId);

    // 3. 다시 조회 → 1차 캐시에서 반환 (likes = 10!)
    Product sameProduct = productRepository.findById(productId);
    // sameProduct.getLikes() → 10 (DB에는 11)
}
```

Native Query는 JPA를 우회하여 DB에 직접 SQL을 실행한다. 1차 캐시는 이 변경을 모른다. 그래서 `@Modifying(clearAutomatically = true)`를 설정하여 **Native Query 실행 후 1차 캐시를 비우도록** 명시하는 것이다.

#### clearAutomatically vs flushAutomatically

`@Modifying`에는 두 가지 옵션이 있다. 이름이 비슷해서 혼동하기 쉽지만, 역할이 완전히 다르다.

```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
```

**`clearAutomatically = true`** — 쿼리 실행 **후** 영속성 컨텍스트를 clear한다.

```java
@Transactional
public void clearExample(Long productId) {
    // 1. 상품 조회 → 1차 캐시에 저장 (likes = 10)
    Product product = productRepository.findById(productId);

    // 2. Native Query 실행 (likes = 11)
    // clearAutomatically = true이므로, 실행 후 1차 캐시가 비워진다
    productRepository.incrementLikeCount(productId);

    // 3. 캐시가 비워졌으므로 DB에서 다시 조회 (likes = 11)
    Product fresh = productRepository.findById(productId);
    // fresh.getLikes() → 11 (최신값!)
}
```

핵심은 **"실행 후(after)"**다. Native Query가 끝나면 1차 캐시를 통째로 비운다. 이후 조회 시 DB에서 최신 데이터를 가져오므로 stale 데이터 문제가 해결된다.

**`flushAutomatically = true`** — 쿼리 실행 **전** 영속성 컨텍스트를 flush한다.

```java
@Transactional
public void flushExample(Long productId) {
    // 1. 상품 조회 → 1차 캐시에 저장
    Product product = productRepository.findById(productId);

    // 2. 엔티티의 이름을 변경 (아직 DB에 반영 안 됨, 1차 캐시에만 존재)
    product.update("새로운 상품명", ...);

    // 3. Native Query 실행
    // flushAutomatically = true이므로, 실행 전에 2번의 변경이 DB에 flush된다
    // flushAutomatically = false라면? 2번의 변경이 DB에 반영되지 않은 채 Native Query가 실행된다
    productRepository.incrementLikeCount(productId);
}
```

핵심은 **"실행 전(before)"**이다. Native Query가 실행되기 전에, 영속성 컨텍스트에 쌓여 있던 변경사항(Dirty 엔티티)을 먼저 DB에 반영한다.

#### 왜 둘 다 필요한가?

`clearAutomatically`만 쓰면, clear 전에 flush되지 않은 변경사항이 유실될 수 있다.

```java
@Transactional
public void dangerousExample(Long productId) {
    Product product = productRepository.findById(productId);

    // 이름 변경 (1차 캐시에만 반영, DB에는 아직 안 감)
    product.update("새로운 상품명", ...);

    // clearAutomatically = true, flushAutomatically = false일 때:
    // → flush 없이 바로 Native Query 실행
    // → Native Query 실행 후 1차 캐시 clear
    // → "새로운 상품명" 변경이 flush도 안 되었는데 캐시에서 사라짐!
    // → 트랜잭션 커밋 시 Dirty Checking 대상이 없으므로 UPDATE도 안 나감
    // → "새로운 상품명" 변경 유실!
    productRepository.incrementLikeCount(productId);
}
```

`flushAutomatically = true`를 함께 쓰면 이 문제가 해결된다.

```java
// 안전한 조합
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("UPDATE products SET likes = likes + 1 WHERE id = :productId", nativeQuery = true)
fun incrementLikeCount(productId: Long)
```

1. `flushAutomatically = true` → Native Query 실행 **전** 변경사항을 DB에 flush
2. Native Query 실행
3. `clearAutomatically = true` → Native Query 실행 **후** 1차 캐시 clear

정리하면 다음과 같다.

| 옵션 | 시점 | 역할 | 없으면? |
|------|------|------|---------|
| `flushAutomatically` | 쿼리 실행 **전** | 쌓인 변경사항을 DB에 반영 | 미반영 변경사항이 유실될 수 있음 |
| `clearAutomatically` | 쿼리 실행 **후** | 1차 캐시를 비워 stale 방지 | 이후 조회 시 stale 데이터 반환 |

> 실무에서는 `clearAutomatically = true`만 써도 대부분 문제없다. Native Query 전에 같은 엔티티를 수정하는 패턴 자체가 드물기 때문이다. 하지만 "혹시 모를 유실"을 방지하려면 `flushAutomatically = true`를 함께 쓰는 것이 안전하다.

---

### 함정 3: JPQL과 1차 캐시

JPQL(또는 Criteria)로 조회할 때도 주의가 필요하다.

```java
@Transactional
public void jpqlTrap() {
    // 1. 상품 조회 → 1차 캐시에 저장 (stock = 100)
    Product product = productRepository.findById(1L);

    // 2. 엔티티 수정 (아직 flush 안 됨, DB에는 100 그대로)
    product.deductStock(Quantity.of(10)); // 메모리상 stock = 90

    // 3. JPQL로 "stock > 95인 상품" 조회
    // JPQL은 DB에 쿼리를 날림 → DB에는 stock = 100이므로 조회됨
    // 하지만 반환할 때 1차 캐시의 엔티티(stock = 90)를 반환!
    List<Product> products = em.createQuery(
        "SELECT p FROM Product p WHERE p.stockQuantity > 95", Product.class
    ).getResultList();

    // products에 stock = 90인 product가 포함됨 (!)
    // DB 조건(stock > 95)에는 맞지만, 메모리 상태(90)와 불일치
}
```

JPQL은 **항상 DB에 쿼리를 실행**한다. 하지만 결과를 반환할 때, 1차 캐시에 이미 같은 ID의 엔티티가 있으면 **DB 결과를 버리고 캐시의 엔티티를 반환**한다. "영속성 컨텍스트의 동일성 보장" 원칙 때문이다.

이로 인해 **WHERE 조건과 실제 반환된 엔티티의 상태가 불일치**하는 상황이 발생할 수 있다.

> 해결책: `em.flush()` + `em.clear()`로 영속성 컨텍스트를 초기화하거나, 비즈니스 로직의 순서를 조정하여 이런 상황 자체를 피하는 것이 좋다.

---

### 1차 캐시 문제 정리

| 상황 | 원인 | 해결 |
|------|------|------|
| 다른 트랜잭션의 변경을 못 봄 | 캐시가 DB 변경을 가림 | 비관적 락으로 최신값 보장 |
| Native Query 후 stale 데이터 | JPA 우회로 캐시 미갱신 | `clearAutomatically = true` |
| JPQL 결과와 캐시 불일치 | 캐시 우선 반환 정책 | `flush()` + `clear()` 또는 순서 조정 |

1차 캐시는 JPA의 핵심 설계이지만, 동시성 환경에서는 **"내가 읽은 데이터가 정말 최신인가?"**를 항상 의심해야 한다. 이 의심이 우리를 **락(Lock)**으로 이끈다.

---

## 낙관적 락(Optimistic Lock)

### 개념: "충돌은 드물 것이다"

낙관적 락은 이름 그대로 **낙관적인 가정**에서 출발한다. "동시에 같은 데이터를 수정하는 일은 드물 것이다." 따라서 **락을 잡지 않고** 작업을 수행하되, 커밋 시점에 충돌을 감지하는 전략이다.

이전 글에서 다룬 **CAS(Compare-And-Swap)**와 원리가 같다.

```
CAS: "현재 값이 내가 읽은 값과 같으면 변경, 다르면 실패"
낙관적 락: "현재 버전이 내가 읽은 버전과 같으면 변경, 다르면 실패"
```

### JPA의 @Version

JPA는 `@Version` 애노테이션으로 낙관적 락을 지원한다. 실제 프로젝트의 주문(Order) 엔티티를 보자.

```kotlin
// Order.kt
@Entity
@Table(name = "orders")
class Order(
    userId: Long,
    status: OrderStatus = OrderStatus.ORDERED,
) : BaseEntity() {

    @Column(name = "user_id", nullable = false)
    var userId: Long = userId
        protected set

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    var status: OrderStatus = status
        protected set

    @Version
    var version: Long = 0
        protected set

    fun changeStatus(next: OrderStatus) {
        if (!status.canTransitionTo(next)) {
            throw CoreException(
                ErrorType.BAD_REQUEST,
                "${status.name}에서 ${next.name}(으)로 상태를 변경할 수 없습니다.",
            )
        }
        this.status = next
    }
}
```

`@Version`이 붙은 `version` 필드가 핵심이다. JPA는 이 엔티티를 UPDATE할 때 자동으로 다음과 같은 SQL을 생성한다.

```sql
UPDATE orders
SET status = 'CONFIRMED', version = 2
WHERE id = 1 AND version = 1
```

**WHERE 절에 version 조건이 추가**된다. 만약 다른 트랜잭션이 이미 version을 올렸다면, 이 UPDATE의 영향 행 수는 0이 되고, JPA는 `OptimisticLockException`을 던진다.

### 동작 흐름

```
시간  트랜잭션A                              트랜잭션B
──────────────────────────────────────────────────────────────────
t1   SELECT * FROM orders WHERE id = 1
     → status = ORDERED, version = 1
t2                                          SELECT * FROM orders WHERE id = 1
                                            → status = ORDERED, version = 1
t3   UPDATE orders SET status = 'CONFIRMED',
     version = 2 WHERE id = 1 AND version = 1
     → 성공! (1행 변경)
t4   COMMIT
t5                                          UPDATE orders SET status = 'DELIVERED',
                                            version = 2 WHERE id = 1 AND version = 1
                                            → 실패! (0행 변경, version이 이미 2)
t6                                          OptimisticLockException 발생!
```

트랜잭션B는 "내가 읽었을 때 version 1이었는데, 지금도 1이면 변경해줘"라고 요청했지만, 이미 version이 2로 올라가 있으므로 실패한다.

### 주문 상태 변경에 낙관적 락을 사용하는 이유

```kotlin
// OrderService.kt
@Transactional
fun changeStatus(orderId: Long, next: OrderStatus) {
    val order = getOrderById(orderId)
    order.changeStatus(next)
}
```

주문 상태 변경은 **경쟁이 드문** 작업이다. 같은 주문의 상태를 동시에 두 곳에서 변경하는 경우는 흔하지 않다. 이런 상황에서는 낙관적 락이 적합하다.

- 대부분의 경우: 충돌 없이 정상 처리
- 드물게 충돌 발생: `OptimisticLockException` → 재시도 또는 에러 반환

비관적 락으로 매번 `SELECT ... FOR UPDATE`를 걸면, 경쟁이 거의 없는데도 **불필요한 락 비용**을 지불하게 된다.

### 낙관적 락의 한계

낙관적 락은 **충돌이 드문 상황**에서만 효과적이다. 충돌이 빈번하면 매번 `OptimisticLockException`이 발생하고, 재시도 로직까지 감안하면 오히려 성능이 나빠진다.

```
경쟁 빈도 낮음 → 낙관적 락 유리 (대부분 성공, 가끔 재시도)
경쟁 빈도 높음 → 낙관적 락 불리 (매번 실패, 매번 재시도)
```

재고 차감처럼 **경쟁이 심한** 작업에는 비관적 락이 필요하다.

---

## 비관적 락(Pessimistic Lock)

### 개념: "충돌이 반드시 발생할 것이다"

비관적 락은 **비관적인 가정**에서 출발한다. "동시에 같은 데이터를 수정하는 일이 빈번할 것이다." 따라서 **데이터를 읽는 시점에 즉시 락을 잡아** 다른 트랜잭션의 접근을 차단한다.

DB의 `SELECT ... FOR UPDATE` 구문을 사용하며, 이전 글의 **`synchronized`**와 원리가 비슷하다.

```
synchronized: 모니터 획득 → 임계 영역 실행 → 모니터 해제
비관적 락: 행 잠금 획득 → 데이터 수정 → 트랜잭션 커밋(잠금 해제)
```

### JPA에서의 비관적 락 — @Lock

실제 프로젝트에서 상품 재고 차감에 비관적 락을 적용한 코드를 보자.

```kotlin
// ProductJpaRepository.kt
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id IN :ids AND p.deletedAt IS NULL ORDER BY p.id")
fun findAllByIdInWithLock(ids: List<Long>): List<Product>
```

`@Lock(LockModeType.PESSIMISTIC_WRITE)`가 핵심이다. JPA는 이 쿼리를 실행할 때 다음과 같은 SQL을 생성한다.

```sql
SELECT p.* FROM products p
WHERE p.id IN (1, 2, 3) AND p.deleted_at IS NULL
ORDER BY p.id
FOR UPDATE
```

**`FOR UPDATE`**가 추가된다. 이 쿼리를 실행한 트랜잭션이 커밋되거나 롤백될 때까지, 해당 행들에 대한 다른 트랜잭션의 쓰기가 차단된다.

### 재고 차감 — 전체 흐름

```kotlin
// OrderFacade.kt
@Transactional
fun placeOrder(userId: Long, items: List<OrderPlaceCommand>, couponId: Long? = null) {
    // 1. 쿠폰 검증 (fail-fast)
    val couponInfo = couponId?.let { id ->
        val issuedCoupon = couponService.findIssuedCouponWithLock(id, userId)
        val coupon = couponService.findCouponById(id)
        issuedCoupon.validateUsable(coupon.expiresAt)
        CouponApplyInfo(id, coupon.discount, issuedCoupon)
    }

    val productIds = items.map { it.productId }

    // 2. 비관적 락으로 상품 조회 (SELECT ... FOR UPDATE)
    val products = productService.getProductsForOrderWithLock(productIds)
    val productMap = products.associateBy { it.id }

    // 3. 브랜드 정보 조회
    val brandMap = brandService.getBrandsByIds(
        products.map { it.brandId }.distinct(),
    ).associateBy { it.id }

    // 4. 재고 차감
    val deductionRequests = items.map { StockDeductionRequest(it.productId, it.quantity) }
    productService.deductStocks(productMap, deductionRequests)

    // 5. 주문 생성
    val order = orderService.createOrder(userId, orderItemCommands)

    // 6. 쿠폰 할인 적용
    couponInfo?.let {
        val discountAmount = it.discount.calculateDiscountAmount(order.totalAmount)
        order.applyCouponDiscount(it.couponId, discountAmount)
        it.issuedCoupon.use()
    }
}
```

핵심은 **2번 단계**다. `getProductsForOrderWithLock`이 비관적 락으로 상품을 조회한다. 이 시점부터 트랜잭션이 끝날 때까지 해당 상품 행은 잠긴다.

```kotlin
// ProductService.kt
fun getProductsForOrderWithLock(productIds: List<Long>): List<Product> {
    val products = productRepository.findAllByIdsWithLock(productIds)
    validateAllProductsFound(productIds, products)
    return products
}

// Product.kt
fun deductStock(quantity: Quantity) {
    this.stockQuantity = this.stockQuantity - quantity
}
```

`deductStock`은 단순히 메모리상의 값을 변경한다. JPA의 **Dirty Checking**이 트랜잭션 커밋 시점에 자동으로 UPDATE 쿼리를 생성한다. 비관적 락이 걸려 있으므로, 다른 트랜잭션이 같은 상품의 재고를 동시에 변경하는 것은 불가능하다.

### 데드락 방지 — ORDER BY

위 코드에서 눈여겨볼 부분이 있다.

```kotlin
@Query("SELECT p FROM Product p WHERE p.id IN :ids AND p.deletedAt IS NULL ORDER BY p.id")
fun findAllByIdInWithLock(ids: List<Long>): List<Product>
```

**`ORDER BY p.id`**가 데드락을 방지한다. 만약 정렬이 없다면?

```
트랜잭션A: 상품1 락 획득 → 상품2 락 대기
트랜잭션B: 상품2 락 획득 → 상품1 락 대기
→ 데드락!
```

ID 오름차순으로 정렬하면 **모든 트랜잭션이 같은 순서로 락을 획득**하므로 데드락이 발생하지 않는다. 이전 글에서 다룬 "락 순서의 일관성"과 같은 원리다.

```
트랜잭션A: 상품1 락 획득 → 상품2 락 획득 → 완료
트랜잭션B: 상품1 락 대기... → 상품1 락 획득 → 상품2 락 획득 → 완료
→ 데드락 없음!
```

### 쿠폰 발급에서의 비관적 락

재고 차감뿐 아니라 쿠폰 발급에서도 비관적 락을 사용한다.

```kotlin
// CouponJpaRepository.kt
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT c FROM Coupon c WHERE c.id = :id AND c.deletedAt IS NULL")
fun findByIdWithLockAndDeletedAtIsNull(id: Long): Coupon?
```

```kotlin
// CouponService.kt
fun issue(couponId: Long, userId: Long) {
    // 비관적 락으로 쿠폰 조회
    val coupon = couponRepository.findByIdWithLock(couponId)
        ?: throw CoreException(ErrorType.NOT_FOUND, "쿠폰을 찾을 수 없습니다.")

    // 중복 발급 체크
    if (issuedCouponRepository.existsByCouponIdAndUserId(couponId, userId)) {
        throw CoreException(ErrorType.CONFLICT, "이미 발급받은 쿠폰입니다.")
    }

    // 수량 차감 + 발급
    coupon.issue()
    issuedCouponRepository.save(IssuedCoupon(couponId = couponId, userId = userId))
}
```

쿠폰의 수량은 제한적이고, 선착순 이벤트 같은 상황에서는 **동시 요청이 폭주**한다. 이런 경우 낙관적 락을 쓰면 대부분 `OptimisticLockException`이 발생하여 재시도만 반복하게 된다. 비관적 락으로 **순서대로 처리**하는 것이 효율적이다.

### 동시성 테스트로 증명하기

실제 프로젝트의 동시성 테스트 코드를 보자.

```kotlin
// StockConcurrencyTest.kt
@DisplayName("동시에 여러 주문이 같은 상품에 들어와도 재고가 정확히 차감된다.")
@Test
fun deductsStockCorrectly_whenConcurrentOrders() {
    // arrange
    val product = createProduct(stockQuantity = StockQuantity.of(100))
    val threadCount = 10
    val executorService = Executors.newFixedThreadPool(threadCount)
    val latch = CountDownLatch(threadCount)
    val successCount = AtomicInteger(0)

    // act
    repeat(threadCount) { i ->
        executorService.submit {
            try {
                orderFacade.placeOrder(
                    userId = i.toLong() + 1,
                    items = listOf(OrderPlaceCommand(productId = product.id, quantity = Quantity.of(1))),
                )
                successCount.incrementAndGet()
            } catch (_: Exception) {
            } finally {
                latch.countDown()
            }
        }
    }
    latch.await()
    executorService.shutdown()

    // assert
    val updatedProduct = productRepository.findById(product.id)
    assertAll(
        { assertThat(successCount.get()).isEqualTo(10) },
        { assertThat(updatedProduct?.stockQuantity).isEqualTo(StockQuantity.of(90)) },
    )
}
```

10개의 스레드가 동시에 같은 상품에 주문을 넣는다. 비관적 락 덕분에 **10개 모두 성공**하고, 재고는 정확히 **100 → 90**으로 차감된다.

재고 초과 테스트도 있다.

```kotlin
@DisplayName("재고보다 많은 동시 주문이 들어오면, 재고가 음수가 되지 않는다.")
@Test
fun doesNotGoNegative_whenConcurrentOrdersExceedStock() {
    val product = createProduct(stockQuantity = StockQuantity.of(5))
    val threadCount = 10
    // ... (동일한 구조)

    assertAll(
        { assertThat(successCount.get()).isEqualTo(5) },
        { assertThat(failCount.get()).isEqualTo(5) },
        { assertThat(updatedProduct?.stockQuantity).isEqualTo(StockQuantity.of(0)) },
    )
}
```

재고 5개에 10개의 동시 주문. 5개만 성공하고, 나머지 5개는 재고 부족으로 실패한다. **재고가 음수가 되지 않는다.** 비관적 락이 순차적 접근을 보장하기 때문이다.

---

## 낙관적 락 vs 비관적 락

### 비교 정리

| 기준 | 낙관적 락 | 비관적 락 |
|------|----------|----------|
| **가정** | 충돌이 드물다 | 충돌이 빈번하다 |
| **잠금 시점** | 커밋 시 충돌 감지 | 조회 시 즉시 잠금 |
| **구현 방식** | `@Version` + 버전 비교 | `@Lock` + `SELECT FOR UPDATE` |
| **충돌 시** | `OptimisticLockException` | 대기 (락 해제까지) |
| **DB 부하** | 낮음 (락 없음) | 높음 (행 잠금) |
| **동시 처리량** | 높음 (충돌 적을 때) | 낮음 (순차 처리) |
| **적합한 상황** | 읽기 위주, 충돌 드문 경우 | 쓰기 위주, 충돌 빈번한 경우 |

### 실전 선택 기준

```
주문 상태 변경 → 낙관적 락 (@Version)
  - 같은 주문을 동시에 변경하는 경우가 드물다
  - 충돌 시 재시도하면 충분하다

재고 차감 → 비관적 락 (SELECT FOR UPDATE)
  - 인기 상품은 동시 주문이 폭주한다
  - 재고는 정확해야 한다 (음수 불가)

쿠폰 발급 → 비관적 락 + Unique Constraint
  - 선착순 이벤트에서 동시 요청이 집중된다
  - 중복 발급 방지가 필수적이다
```

---

## 마무리

이번 글에서 다룬 내용을 정리하면 다음과 같다.

> **DB 격리 수준**
> - Dirty Read: 커밋 안 된 데이터를 읽는 현상 → READ COMMITTED 이상에서 방지
> - Non-Repeatable Read: 같은 행의 값이 달라지는 현상 → REPEATABLE READ 이상에서 방지
> - Phantom Read: 행의 개수가 달라지는 현상 → SERIALIZABLE에서 방지 (MySQL은 REPEATABLE READ에서도 대부분 방지)

> **JPA 1차 캐시의 함정**
> - 다른 트랜잭션의 변경을 가린다
> - Native Query는 캐시를 우회한다 → `clearAutomatically = true`
> - JPQL 결과와 캐시가 불일치할 수 있다

> **낙관적 락 vs 비관적 락**
> - 낙관적 락: `@Version`으로 커밋 시 충돌 감지. 경쟁이 드문 상황에 적합
> - 비관적 락: `SELECT FOR UPDATE`로 즉시 잠금. 경쟁이 심한 상황에 적합
> - 데드락 방지: 락 획득 순서를 일관되게 유지 (`ORDER BY id`)

이전 글의 Java 동시성 도구와 이번 글의 DB/JPA 동시성 도구를 비교하면 흥미로운 대응 관계가 보인다.

| Java 도구 | DB/JPA 대응 |
|-----------|-------------|
| `synchronized` / `ReentrantLock` | 비관적 락 (`SELECT FOR UPDATE`) |
| CAS / `AtomicInteger` | 낙관적 락 (`@Version`) |
| `volatile` | 1차 캐시 clear / refresh |

결국 레벨만 다를 뿐, **"공유 자원에 대한 동시 접근을 어떻게 제어할 것인가"**라는 근본 질문은 같다.

다음 편에서는 **DB Lock**의 세계로 더 깊이 들어간다. MySQL의 Row Lock, Gap Lock, Next-Key Lock, 그리고 트랜잭션 격리 수준과 락의 관계를 다룰 예정이다.

---

## 참고자료

- Vlad Mihalcea, 「High-Performance Java Persistence」
- Oracle, [JPA Locking](https://docs.oracle.com/javaee/7/tutorial/persistence-locking.html)
- MySQL 공식 문서, [InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
