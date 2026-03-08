# 동시성과 Lock의 개념 - MySQL Lock편

## 들어가며

[이전 글](https://zzangoobrother.github.io/BLOG/?post=%5B20260306%5D_%5B%EB%8F%99%EC%8B%9C%EC%84%B1%EA%B3%BC+Lock%EC%9D%98+%EA%B0%9C%EB%85%90+-+DB%EC%99%80+JPA%ED%8E%B8%5D_%5B%EB%8F%99%EC%8B%9C%EC%84%B1%5D_%5B%5D_%5BDB+%EA%B2%A9%EB%A6%AC+%EC%88%98%EC%A4%80%2C+JPA+%EC%BA%90%EC%8B%9C+%EC%A0%95%ED%95%A9%EC%84%B1+%EB%AC%B8%EC%A0%9C%2C+%EB%82%99%EA%B4%80%EC%A0%81+%EB%9D%BD%EA%B3%BC+%EB%B9%84%EA%B4%80%EC%A0%81+%EB%9D%BD%5D_%5B%5D.md)에서는 DB 격리 수준, JPA 1차 캐시의 함정, 낙관적 락과 비관적 락을 다뤘다. `SELECT FOR UPDATE`가 "행을 잠근다"는 것까지는 이해했지만, **MySQL이 정확히 무엇을 어떻게 잠그는지**는 다루지 않았다.

이번 글에서는 MySQL InnoDB 스토리지 엔진의 내부 락 메커니즘을 깊이 파고든다.

1. InnoDB의 락 종류 — Record Lock, Gap Lock, Next-Key Lock
2. 격리 수준에 따라 락이 어떻게 달라지는가
3. 실전에서 이 지식이 왜 필요한가

---

## InnoDB의 락 아키텍처

### 먼저 알아야 할 것: 인덱스 기반 잠금

InnoDB의 락을 이해하는 핵심은 한 가지다.

> **InnoDB는 행(Row)을 잠그는 것이 아니라, 인덱스를 잠근다.**

이것을 모르면 이후의 모든 설명이 이해되지 않는다. `SELECT * FROM products WHERE id = 1 FOR UPDATE`를 실행하면, `id = 1`인 **행**이 잠기는 게 아니라 `id = 1`에 해당하는 **인덱스 레코드**가 잠긴다.

인덱스가 없는 컬럼으로 조건을 걸면? InnoDB는 **테이블 전체를 스캔하면서 모든 인덱스 레코드에 락을 건다.** 사실상 테이블 락과 같은 효과다. 이것이 `WHERE` 조건에 인덱스가 중요한 또 다른 이유다.

---

## Record Lock — 인덱스 레코드 잠금

### 개념

**Record Lock**은 가장 기본적인 행 수준 잠금이다. **인덱스 레코드 하나**를 잠근다.

```sql
-- products 테이블 (id는 PRIMARY KEY)
-- id: 1, 3, 5, 7, 10

SELECT * FROM products WHERE id = 5 FOR UPDATE;
```

이 쿼리는 `id = 5`인 인덱스 레코드에 **Record Lock**을 건다. 다른 트랜잭션은 `id = 5`인 행을 수정하거나 삭제할 수 없지만, `id = 3`이나 `id = 7`은 자유롭게 수정할 수 있다.

```
인덱스:  1    3    5    7    10
         ○    ○    ●    ○    ○

● = Record Lock (잠김)
○ = 잠기지 않음
```

### 실전: 단일 상품 조회에 비관적 락 적용

커머스 프로젝트에서 단일 상품에 비관적 락을 거는 코드를 보자.

```kotlin
// ProductJpaRepository.kt
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id = :id AND p.deletedAt IS NULL")
fun findByIdWithLock(id: Long): Product?
```

JPA가 생성하는 SQL은 다음과 같다.

```sql
SELECT p.* FROM products p
WHERE p.id = 1 AND p.deleted_at IS NULL
FOR UPDATE
```

`id`는 PRIMARY KEY이므로 유니크 인덱스에 대한 **등호(=) 조건**이다. 이 경우 InnoDB는 **Record Lock만** 건다. 정확히 해당 레코드 하나만 잠기므로 다른 상품에는 영향을 주지 않는다.

> **핵심 규칙:** 유니크 인덱스에 대한 등호 조건은 Record Lock만 건다. Gap Lock이 불필요하기 때문이다 — 유니크 인덱스에는 같은 값이 두 개 존재할 수 없으므로, 간격(gap)을 잠글 이유가 없다.

---

## Gap Lock — 간격 잠금

### 개념

**Gap Lock**은 인덱스 레코드 **사이의 간격(gap)**을 잠근다. 레코드 자체가 아니라 **레코드와 레코드 사이의 빈 공간**을 잠그는 것이다.

```sql
-- products 테이블의 인덱스 상태
-- id: 1, 3, 5, 7, 10

SELECT * FROM products WHERE id = 4 FOR UPDATE;
```

`id = 4`인 레코드는 존재하지 않는다. 이 경우 InnoDB는 `id = 4`가 **들어갈 수 있는 간격**, 즉 `(3, 5)` 구간에 Gap Lock을 건다.

```
인덱스:  1    3    [   gap   ]    5    7    10
         ○    ○    ◄━━━━━━━━━►    ○    ○    ○

◄━━━━━━━━━► = Gap Lock (이 간격에 INSERT 불가)
```

Gap Lock이 걸리면 다른 트랜잭션은 `id = 4`를 INSERT할 수 없다. 하지만 `id = 3`이나 `id = 5`를 수정하는 것은 허용된다. **Gap Lock은 INSERT만 차단**한다.

### 왜 간격을 잠글까?

Gap Lock의 존재 이유는 **Phantom Read 방지**다. 이전 글에서 다뤘던 Phantom Read를 다시 떠올려보자.

```
트랜잭션A: SELECT * FROM products WHERE price BETWEEN 1000 AND 2000 → 3건
트랜잭션B: INSERT INTO products (price) VALUES (1500) → 성공
트랜잭션A: SELECT * FROM products WHERE price BETWEEN 1000 AND 2000 → 4건 (!)
```

Gap Lock이 없으면, 트랜잭션A가 범위 조회를 하는 동안 트랜잭션B가 그 범위에 새로운 행을 삽입할 수 있다. Gap Lock은 **"이 간격에는 아무도 INSERT하지 마라"**고 선언하여 Phantom Read를 방지한다.

### Gap Lock의 특성

Gap Lock에는 독특한 특성이 있다.

1. **Gap Lock끼리는 충돌하지 않는다.** 두 트랜잭션이 같은 간격에 Gap Lock을 동시에 걸 수 있다. 둘 다 INSERT를 차단하는 "방어적 잠금"이기 때문이다.
2. **Gap Lock은 INSERT만 차단한다.** UPDATE나 DELETE는 차단하지 않는다.
3. **READ COMMITTED에서는 Gap Lock이 비활성화된다.** Gap Lock은 Phantom Read 방지 목적이므로, Phantom Read를 허용하는 READ COMMITTED에서는 불필요하다.

---

## Next-Key Lock — Record Lock + Gap Lock

### 개념

**Next-Key Lock**은 **Record Lock과 Gap Lock의 조합**이다. 인덱스 레코드 자체와 그 레코드 **앞의 간격**을 함께 잠근다.

```sql
-- products 테이블의 인덱스 상태
-- id: 1, 3, 5, 7, 10

SELECT * FROM products WHERE id BETWEEN 3 AND 7 FOR UPDATE;
```

이 범위 조회에서 InnoDB는 **Next-Key Lock**을 사용한다.

```
인덱스:  1    3         5         7         10
         ○    ●━━━━━━━━━●━━━━━━━━━●━━━━━━━━━►
              ↑                              ↑
              Record + Gap Lock              Gap Lock (7 다음 간격까지)

잠기는 범위: (1, 3], (3, 5], (5, 7], (7, 10)
```

Next-Key Lock은 **(이전 레코드, 현재 레코드]** 형태의 반개방 구간을 잠근다. 위 예시에서는 `id = 3, 5, 7`의 레코드와, 그 사이의 모든 간격, 그리고 `id = 7` 다음의 간격까지 잠긴다.

### 왜 레코드 뒤의 간격까지 잠글까?

범위 조건의 끝 부분에서 새로운 행이 삽입되는 것을 막기 위해서다.

```sql
-- Next-Key Lock이 (7, 10) 간격까지 잠그지 않는다면?
-- 다른 트랜잭션이 id = 8을 INSERT할 수 있다.
-- 그러면 WHERE id BETWEEN 3 AND 7의 결과에는 영향이 없지만,
-- WHERE id <= 7 같은 조건에서는 Phantom이 발생할 수 있다.
```

### Next-Key Lock이 Record Lock으로 축소되는 경우

앞서 Record Lock에서 언급했듯, **유니크 인덱스에 대한 등호 조건**에서는 Next-Key Lock이 **Record Lock으로 축소**된다.

```sql
-- 유니크 인덱스(PK) + 등호 조건 → Record Lock만
SELECT * FROM products WHERE id = 5 FOR UPDATE;

-- 유니크 인덱스 + 범위 조건 → Next-Key Lock
SELECT * FROM products WHERE id BETWEEN 3 AND 7 FOR UPDATE;

-- 비유니크 인덱스 + 등호 조건 → Next-Key Lock
SELECT * FROM products WHERE brand_id = 1 FOR UPDATE;
```

왜 유니크 인덱스 등호 조건에서만 축소될까? 유니크 인덱스에 `id = 5`가 있으면, `id = 5`인 행이 **정확히 하나만 존재**한다는 것이 보장된다. 간격을 잠그지 않아도 Phantom Read가 발생할 수 없다. 하지만 비유니크 인덱스에서 `brand_id = 1`은 여러 행이 존재할 수 있으므로, 간격까지 잠가야 새로운 `brand_id = 1` 행의 삽입을 막을 수 있다.

---

## 세 가지 락 비교

| 락 종류 | 잠그는 대상 | 차단하는 연산 | 사용 시점 |
|---------|-----------|-------------|----------|
| **Record Lock** | 인덱스 레코드 하나 | UPDATE, DELETE | 유니크 인덱스 등호 조건 |
| **Gap Lock** | 레코드 사이의 간격 | INSERT | 존재하지 않는 값 조회, 범위 조건 |
| **Next-Key Lock** | 레코드 + 앞의 간격 | UPDATE, DELETE, INSERT | 비유니크 인덱스, 범위 조건 (기본값) |

> InnoDB의 **기본 잠금 단위는 Next-Key Lock**이다. 특정 조건(유니크 인덱스 등호)을 만족하면 Record Lock으로 축소된다.

---

## 격리 수준과 락의 관계

### READ COMMITTED — Gap Lock 없음

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- id: 1, 3, 5, 7, 10
SELECT * FROM products WHERE id BETWEEN 3 AND 7 FOR UPDATE;
```

READ COMMITTED에서는 **Gap Lock이 비활성화**된다. 위 쿼리는 `id = 3, 5, 7`에 Record Lock만 건다. 간격은 잠기지 않으므로 `id = 4`나 `id = 6`을 INSERT할 수 있다.

```
READ COMMITTED에서의 잠금:

인덱스:  1    3    [열림]    5    [열림]    7    10
         ○    ●              ●              ●    ○

● = Record Lock만 (Gap 없음)
```

**장점:** 잠금 범위가 좁아 동시 처리량이 높다.
**단점:** Phantom Read가 발생할 수 있다.

### REPEATABLE READ — Next-Key Lock (InnoDB 기본값)

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- id: 1, 3, 5, 7, 10
SELECT * FROM products WHERE id BETWEEN 3 AND 7 FOR UPDATE;
```

REPEATABLE READ에서는 **Next-Key Lock**이 기본이다. `id = 3, 5, 7`의 레코드와 그 사이의 간격이 모두 잠긴다.

```
REPEATABLE READ에서의 잠금:

인덱스:  1    3━━━━━━━━━5━━━━━━━━━7━━━━━━━━━►    10
         ○    ●━━━━━━━━━●━━━━━━━━━●━━━━━━━━━      ○

●━━━━━━━━━ = Next-Key Lock (레코드 + 간격)
```

**장점:** Phantom Read가 방지된다. INSERT가 차단되므로 범위 조회 결과가 트랜잭션 내에서 변하지 않는다.
**단점:** 잠금 범위가 넓어 동시 처리량이 떨어질 수 있다.

### SERIALIZABLE — 모든 SELECT가 FOR SHARE

SERIALIZABLE에서는 일반 `SELECT`도 **자동으로 `SELECT ... FOR SHARE`(공유 잠금)**로 변환된다. 읽기 작업도 잠금을 획득하므로, 완전한 직렬화가 보장된다.

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- 이 SELECT도 공유 잠금이 걸린다
SELECT * FROM products WHERE price > 1000;
-- → SELECT * FROM products WHERE price > 1000 FOR SHARE; 와 동일
```

실무에서 SERIALIZABLE을 쓰는 경우는 거의 없다. 성능이 극도로 떨어지기 때문이다.

### 격리 수준별 잠금 동작 정리

| 격리 수준 | 일반 SELECT | SELECT FOR UPDATE | Gap Lock |
|-----------|-----------|-------------------|----------|
| READ UNCOMMITTED | 잠금 없음 | Record Lock | X |
| READ COMMITTED | 잠금 없음 (스냅샷) | Record Lock | X |
| REPEATABLE READ | 잠금 없음 (스냅샷) | Next-Key Lock | O |
| SERIALIZABLE | 공유 잠금 (FOR SHARE) | Next-Key Lock | O |

---

## 실전에서 이 지식이 필요한 순간

### 사례 1: 비유니크 인덱스로 FOR UPDATE를 걸 때

커머스 프로젝트에서 브랜드별 상품을 조회하며 락을 건다고 가정해보자.

```sql
-- brand_id는 비유니크 인덱스
-- brand_id: 1(상품A), 1(상품B), 2(상품C), 3(상품D)

SELECT * FROM products WHERE brand_id = 1 FOR UPDATE;
```

이 쿼리는 `brand_id = 1`인 두 레코드에 Record Lock을 걸 뿐 아니라, **`brand_id = 1`이 존재하는 간격에도 Gap Lock**을 건다. 다른 트랜잭션은 `brand_id = 1`인 새 상품을 INSERT할 수 없다.

이것이 의도한 동작이라면 좋지만, 단순히 "기존 상품만 잠그고 싶었다"면 **예상보다 넓은 범위가 잠긴** 것이다. 동시에 `brand_id = 1`인 신상품을 등록하려는 어드민 요청이 대기하게 될 수 있다.

### 사례 2: 왜 PK로 잠그는 것이 효율적인가

프로젝트의 재고 차감 쿼리를 다시 보자.

```kotlin
// ProductJpaRepository.kt
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id IN :ids AND p.deletedAt IS NULL ORDER BY p.id")
fun findAllByIdInWithLock(ids: List<Long>): List<Product>
```

`id`는 PRIMARY KEY(유니크 인덱스)이고, `IN` 조건은 내부적으로 **각 값에 대한 등호 조건**으로 처리된다. 따라서 각 `id`에 대해 **Record Lock만** 걸린다.

```sql
-- id IN (1, 3, 5)에 대한 잠금
-- id: 1, 2, 3, 4, 5, 6, 7 ...

인덱스:  1    2    3    4    5    6    7
         ●    ○    ●    ○    ●    ○    ○

● = Record Lock만 (Gap 없음)
```

간격이 잠기지 않으므로 `id = 2`나 `id = 4`에 대한 작업은 자유롭다. **잠금 범위가 최소화**된다. 만약 `brand_id`(비유니크) 기준으로 잠갔다면 간격까지 잠겨서 불필요한 대기가 발생했을 것이다.

> PK 기반 비관적 락은 Record Lock만 걸리므로, **잠금의 영향 범위가 가장 좁다.** 이것이 비관적 락을 걸 때 PK 기반 조회를 사용하는 실전적인 이유다.

### 사례 3: ORDER BY로 데드락을 방지하는 원리

프로젝트에서 `ORDER BY p.id`로 데드락을 방지하는 이유를 이제 락 수준에서 설명할 수 있다.

```sql
-- 정렬 없이 락을 획득하는 경우

-- 트랜잭션A: SELECT ... WHERE id IN (1, 5) FOR UPDATE
-- DB가 id = 5를 먼저 잠그고, id = 1을 나중에 잠글 수 있다

-- 트랜잭션B: SELECT ... WHERE id IN (1, 5) FOR UPDATE
-- DB가 id = 1을 먼저 잠그고, id = 5를 나중에 잠글 수 있다

-- 결과: A가 5의 Record Lock을 잡고 1을 대기, B가 1의 Record Lock을 잡고 5를 대기 → 데드락
```

`ORDER BY id`를 명시하면 InnoDB가 **항상 인덱스 순서대로** Record Lock을 획득한다.

```sql
-- 정렬 있이 락을 획득하는 경우

-- 트랜잭션A: SELECT ... WHERE id IN (1, 5) ORDER BY id FOR UPDATE
-- id = 1 Record Lock → id = 5 Record Lock

-- 트랜잭션B: SELECT ... WHERE id IN (1, 5) ORDER BY id FOR UPDATE
-- id = 1 Record Lock 대기... → (A 완료 후) id = 1 → id = 5

-- 결과: 순차 처리, 데드락 없음
```

### 사례 4: 존재하지 않는 행에 FOR UPDATE를 걸면?

쿠폰 발급 시, 발급 이력이 존재하지 않는 경우를 생각해보자.

```sql
-- issued_coupons 테이블
-- id: 1, 5, 10

SELECT * FROM issued_coupons WHERE id = 3 FOR UPDATE;
-- id = 3은 존재하지 않는다
```

레코드가 없으므로 Record Lock을 걸 수 없다. 이 경우 InnoDB는 **(1, 5) 간격에 Gap Lock**을 건다. 다른 트랜잭션이 `id = 2, 3, 4`를 INSERT하는 것이 차단된다.

```
인덱스:  1    [━━ Gap Lock ━━]    5    10
         ○    ◄━━━━━━━━━━━━━━►    ○    ○
```

이것은 의도치 않은 부작용이 될 수 있다. "존재하지 않으면 INSERT" 패턴에서 **두 트랜잭션이 동시에 같은 값을 조회하면, 둘 다 Gap Lock을 획득**한다 (Gap Lock끼리는 호환). 이후 둘 다 INSERT를 시도하면, 상대방의 Gap Lock 때문에 **데드락**이 발생할 수 있다.

```
트랜잭션A: SELECT WHERE id = 3 FOR UPDATE → Gap Lock (1, 5) 획득
트랜잭션B: SELECT WHERE id = 3 FOR UPDATE → Gap Lock (1, 5) 획득 (호환됨!)
트랜잭션A: INSERT id = 3 → B의 Gap Lock 때문에 대기
트랜잭션B: INSERT id = 3 → A의 Gap Lock 때문에 대기
→ 데드락!
```

이 패턴이 커머스 프로젝트에서 쿠폰 중복 발급 방지에 **UNIQUE 제약**을 사용하는 이유이기도 하다. "조회 후 없으면 INSERT" 대신, **일단 INSERT하고 UNIQUE 위반 시 예외를 처리**하는 방식이 더 안전하다.

---

## Insert Intention Lock — INSERT 전용 Gap Lock

INSERT가 Gap Lock에 의해 차단될 때, InnoDB는 **Insert Intention Lock**이라는 특수한 Gap Lock을 사용한다. 이것은 "나는 이 간격에 INSERT할 의향이 있다"는 신호다.

```sql
-- id: 1, 5, 10
-- (1, 5) 간격에 Gap Lock이 걸려있는 상태

-- 트랜잭션C: INSERT INTO products (id, ...) VALUES (3, ...);
-- → (1, 5) 간격에 대한 Insert Intention Lock을 요청
-- → 기존 Gap Lock과 충돌 → 대기
```

Insert Intention Lock의 핵심 특성은 **같은 간격이라도 서로 다른 위치에 INSERT하는 트랜잭션끼리는 충돌하지 않는다**는 것이다.

```sql
-- (1, 10) 간격에 대해
-- 트랜잭션A: INSERT id = 3  →  Insert Intention Lock
-- 트랜잭션B: INSERT id = 7  →  Insert Intention Lock
-- → 서로 다른 위치이므로 동시에 진행 가능!
```

이 설계 덕분에 같은 간격에 대한 INSERT가 불필요하게 직렬화되지 않는다.

---

## 락 확인 방법 — performance_schema

실제로 어떤 락이 걸렸는지 확인하고 싶다면 `performance_schema`를 사용한다.

```sql
-- 현재 걸려있는 락 조회
SELECT
    ENGINE_LOCK_ID,
    ENGINE_TRANSACTION_ID,
    LOCK_TYPE,      -- TABLE 또는 RECORD
    LOCK_MODE,      -- X (배타적), S (공유적), X,GAP, X,REC_NOT_GAP 등
    LOCK_STATUS,    -- GRANTED 또는 WAITING
    INDEX_NAME,
    OBJECT_NAME,
    LOCK_DATA       -- 잠긴 인덱스 레코드 값
FROM performance_schema.data_locks
WHERE OBJECT_SCHEMA = 'commerce';
```

```sql
-- 락 대기 관계 확인
SELECT
    REQUESTING_ENGINE_TRANSACTION_ID AS waiting_trx,
    BLOCKING_ENGINE_TRANSACTION_ID AS blocking_trx,
    REQUESTING_ENGINE_LOCK_ID AS waiting_lock,
    BLOCKING_ENGINE_LOCK_ID AS blocking_lock
FROM performance_schema.data_lock_waits;
```

### LOCK_MODE 읽는 법

| LOCK_MODE | 의미 |
|-----------|------|
| `X,REC_NOT_GAP` | Record Lock (배타적) |
| `X,GAP` | Gap Lock (배타적) |
| `X` | Next-Key Lock (배타적) |
| `S,REC_NOT_GAP` | Record Lock (공유적) |
| `S,GAP` | Gap Lock (공유적) |
| `S` | Next-Key Lock (공유적) |

예를 들어 PK 기반 `SELECT FOR UPDATE`를 실행하면 `X,REC_NOT_GAP`이 보인다. 범위 조건이면 `X`(Next-Key Lock)가 보인다.

---

## 정리: 언제 어떤 락이 걸리는가

| 조건 | 격리 수준 | 걸리는 락 |
|------|----------|----------|
| 유니크 인덱스 + 등호 (값 존재) | REPEATABLE READ | Record Lock |
| 유니크 인덱스 + 등호 (값 미존재) | REPEATABLE READ | Gap Lock |
| 유니크 인덱스 + 범위 조건 | REPEATABLE READ | Next-Key Lock |
| 비유니크 인덱스 + 등호 | REPEATABLE READ | Next-Key Lock |
| 비유니크 인덱스 + 범위 조건 | REPEATABLE READ | Next-Key Lock |
| 인덱스 없음 | REPEATABLE READ | 모든 레코드에 Next-Key Lock (≈ 테이블 락) |
| 모든 조건 | READ COMMITTED | Record Lock만 (Gap Lock 비활성화) |

---

## 마무리

MySQL InnoDB의 락은 단순히 "행을 잠근다"가 아니다. **인덱스를 기반으로 레코드와 간격을 잠그며, 격리 수준에 따라 잠금 범위가 달라진다.**

이번 시리즈를 통해 동시성 제어를 세 단계로 내려가며 살펴봤다.

| 레벨 | 도구 | 제어 대상 |
|------|------|----------|
| **Java** | `synchronized`, `ReentrantLock`, CAS | 메모리상의 공유 변수 |
| **JPA/애플리케이션** | `@Version`, `@Lock`, `SELECT FOR UPDATE` | 엔티티/행 단위의 데이터 |
| **MySQL 내부** | Record Lock, Gap Lock, Next-Key Lock | 인덱스 레코드와 간격 |

커머스 프로젝트에서 **PK 기반으로 비관적 락을 걸고, ORDER BY로 데드락을 방지**하는 패턴이 왜 효과적인지, 이제 MySQL 내부 잠금 수준에서 설명할 수 있다.

- PK(유니크 인덱스) 등호 조건 → **Record Lock만** 걸린다. 잠금 범위가 최소화된다.
- ORDER BY id → **인덱스 순서대로 Record Lock을 획득**한다. 교차 잠금이 발생하지 않는다.
- 비유니크 인덱스를 피한다 → **불필요한 Gap Lock이 발생하지 않는다.** 다른 트랜잭션의 INSERT가 불필요하게 차단되지 않는다.

> "행을 잠근다"는 한 마디 뒤에는, 인덱스 구조와 격리 수준과 간격 잠금이라는 세계가 숨어있다. 그 세계를 이해하면, `SELECT FOR UPDATE` 한 줄을 쓸 때도 **정확히 무엇이 잠기고, 무엇이 대기하는지** 예측할 수 있다.

---

## 참고자료

- MySQL 공식 문서, [InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
- MySQL 공식 문서, [Locks Set by Different SQL Statements](https://dev.mysql.com/doc/refman/8.0/en/innodb-locks-set.html)
- Vlad Mihalcea, 「High-Performance Java Persistence」
- 이성욱, 「Real MySQL 8.0」
