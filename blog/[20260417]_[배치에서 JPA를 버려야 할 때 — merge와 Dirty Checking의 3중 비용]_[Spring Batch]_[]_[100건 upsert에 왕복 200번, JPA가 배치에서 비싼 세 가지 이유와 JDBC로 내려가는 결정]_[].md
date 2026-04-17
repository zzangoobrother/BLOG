# 배치에서 JPA를 버려야 할 때 — merge와 Dirty Checking의 3중 비용

## 들어가며

주간 상품 랭킹 배치를 만든다고 하자. 매일 새벽 3시, `product_metrics_daily` 테이블에서 이번 주 7일치 집계를 읽어 `mv_product_rank_weekly`(주간 TOP 100 Materialized View)에 upsert한다. 프로젝트는 JPA 기반이고, 다른 Repository도 모두 `JpaRepository`를 쓰고 있다. 그래서 자연스럽게 `JpaItemWriter`를 고른다.

돌려봤더니 10분이 걸렸다. 같은 로직을 `JdbcBatchItemWriter + ON DUPLICATE KEY UPDATE`로 바꿨더니 30초로 줄었다. **20배.**

단순히 "JPA가 느리다"로 끝낼 일이 아니다. JPA는 도메인 로직에서는 훌륭한 도구고, 같은 팀이 같은 DB를 상대로 같은 쿼리를 날리는데 20배 차이가 난다면 거기엔 이유가 있다. 그리고 그 이유는 대부분 **"JPA의 장점이 배치에서는 비용으로 뒤집히기 때문"**이다.

이 글에서는 세 가지를 다룬다.

1. 100건 upsert가 왜 200번의 DB 왕복이 되는가 — `merge`의 선행 SELECT와 batching이 원리적으로 불가능한 지점
2. 읽기 전용 배치가 왜 메모리를 2배 쓰는가 — Dirty Checking 스냅샷의 비용
3. JDBC로 내려가는 결정이 **"프로젝트 일관성 훼손"**이 아닌 이유 — 일관성의 대상을 재정의하기

---

## 장면 — 100건 upsert, 왕복 200번

### JpaItemWriter의 동작

Spring Batch에서 upsert를 JPA로 구현하면 `EntityManager.merge(entity)`가 호출된다. `merge`의 동작을 한 식별자 단위로 뜯어보면 이렇다.

```
1. 식별자로 영속성 컨텍스트(1차 캐시)에 존재 여부 확인
2. 없으면 DB에 SELECT로 조회
3. DB에 있으면 → 영속 상태로 attach, 변경 필드는 Dirty Checking으로 UPDATE 생성
4. DB에 없으면 → 새 Entity로 간주, INSERT 생성
```

핵심은 **"판단을 위해 DB SELECT가 먼저 일어난다"**는 점이다. chunk에 100건이 있으면 식별자 100개마다 이 판단이 반복된다. SELECT 100번, 그 다음 INSERT 또는 UPDATE 100번. 합계 200번의 왕복.

```
chunk size = 100
  → SELECT × 100 (식별자별)
  → INSERT 또는 UPDATE × 100
  → 총 ~200 왕복
```

INSERT/UPDATE는 Hibernate batch 옵션(`hibernate.jdbc.batch_size`)으로 묶을 수 있다. 하지만 **SELECT는 묶을 수 없다.** 왜 그럴까.

### SELECT가 batching되지 않는 이유

100건의 SELECT를 `WHERE id IN (...)`으로 한 번에 묶을 수는 없다. 왜냐하면 다음 로직이 SELECT 결과에 **순차적으로 의존**하기 때문이다.

```
식별자 1 SELECT → 존재? → persist or UPDATE 결정 → flush
식별자 2 SELECT → 존재? → persist or UPDATE 결정 → flush
...
```

`merge`는 각 식별자마다 **"지금 이게 신규인가 기존인가"**를 판단하고, 그 판단에 따라 Entity 상태를 전이시킨다. 상태 전이는 Hibernate가 관리하는 내부 머신이고, 이 머신은 한 건씩 돌아갈 수밖에 없다. 100건을 한 번에 조회해서 메모리에 올려두고 분기하는 최적화는 JPA의 Entity 상태 머신 설계상 불가능하다.

**JPA는 "한 Entity씩 생명주기를 정교하게 관리"하는 데 최적화되어 있다. 배치처럼 "100건을 한 번에 쏟아붓는" 상황은 JPA의 설계 철학과 반대편에 있다.**

---

## 비용 1 — Dirty Checking, 읽기 전용인데도 메모리 2배

merge만 비싼 게 아니다. Reader 쪽의 영속성 컨텍스트도 숨은 비용을 낸다.

### 스냅샷이라는 그림자

영속성 컨텍스트는 Entity를 받을 때마다 **스냅샷(Snapshot)**을 별도로 보관한다. 스냅샷은 Entity 필드 값의 복사본이다.

```
영속성 컨텍스트 안:
  Entity(id=1, view=100, like=30)       ← 현재 상태
  Snapshot(id=1, view=100, like=30)     ← 로드 시점의 복사본
```

flush 또는 commit 시점에 현재 Entity와 스냅샷을 **필드 단위로 비교**한다. 차이가 있는 필드에 대해서만 UPDATE를 자동 생성한다. 이것이 Dirty Checking(변경 감지)이다.

도메인 로직에서는 이 동작이 강력하다. `entity.setStatus(PAID)` 한 줄이면 SQL을 쓰지 않아도 UPDATE가 나간다. 개발자가 "어떤 필드가 바뀌었는지" 추적할 필요가 없다.

### 배치에서는 전부 낭비다

집계 배치의 Reader는 어떤가? `product_metrics_daily`에서 `(product_id, SUM(view), SUM(like), SUM(sales))`를 읽어 Processor에 넘겨주고, 그게 끝이다. **Entity 상태를 0% 수정한다.** 그런데 JPA는:

- 스냅샷을 **100% 저장한다** → 메모리 2배 (Entity + Snapshot)
- commit 시점에 **필드별 비교 루프를 무조건 돈다** → CPU 낭비
- 200만 row × 2000 chunks → 수백만 번의 필드 비교 + Young GC 압박

해결책은 있다. `@QueryHints(HINT_READONLY=true)` 또는 `Session.setReadOnly(true)`로 스냅샷 생성을 생략할 수 있다.

하지만 여기서 질문이 생긴다.

> **영속성 컨텍스트의 핵심 장점(Dirty Checking)을 끄고 JPA를 쓴다는 것은, JPA를 쓸 이유가 없다는 신호 아닌가?**

이 질문이 뒤에서 다시 돌아온다.

---

## 비용 2 — 트랜잭션 경계가 설정에 달려있다

JpaPagingItemReader는 `transacted`라는 옵션을 가진다. 기본값은 `true`다.

```
transacted = true  (기본값)
  → Reader가 자체 트랜잭션으로 페이지를 읽고 즉시 커밋
  → Entity는 chunk 트랜잭션에 detached 상태로 전달
  → chunk 트랜잭션과 다른 트랜잭션에 속함

transacted = false
  → Reader/Writer가 chunk 트랜잭션 하나를 공유
  → Entity가 영속 상태로 chunk 내내 체류
```

여기에 `pageSize`와 `chunkSize`의 관계가 또 섞인다. `pageSize > chunkSize`면 다음 chunk용 Entity가 미리 영속성 컨텍스트에 올라와 누적된다. 200만 건 Reader에서 이 설정을 하나 삐끗하면 메모리 누수다.

**"교과서대로 설정했을 때만 메모리 문제가 없다."** 이 말은 실무에서 위험 신호다. 설정의 조합이 트랜잭션 경계와 메모리 수명을 좌우하고, 디버깅할 때 "영속성 컨텍스트가 지금 어느 상태인가"를 추적해야 한다. JDBC에는 이 층이 아예 없다.

> JPA의 **"똑똑한 관리"**가 배치에서는 **"디버깅해야 할 또 하나의 레이어"**가 된다.

---

## 비용 3 — Reader 쿼리가 애초에 Entity가 아니다

주간 랭킹 배치의 Reader가 실행하는 쿼리를 보자.

```sql
SELECT product_id,
       SUM(view_count)  AS view_count,
       SUM(like_count)  AS like_count,
       SUM(sales_count) AS sales_count
FROM product_metrics_daily
WHERE metric_date BETWEEN ? AND ?
GROUP BY product_id
```

결과 row는 4개 컬럼이다. `(product_id, SUM(view), SUM(like), SUM(sales))`.

반면 `ProductMetricsDaily` Entity는 `(id, product_id, metric_date, view_count, like_count, sales_count, created_at, updated_at, ...)` 구조. **컬럼 개수도 다르고, `metric_date`는 GROUP BY로 소거되어 사라졌다.**

이 결과는 Entity가 아니다. 일시적인 집계 DTO일 뿐이다. JPA로 우회하려면 `SELECT new ProductAggregateDto(...)` 생성자 투영을 쓰면 되지만:

- Entity 매핑 0%
- 영속성 컨텍스트 불필요
- 연관관계 매핑 0%
- 1차 캐시 혜택 0%

JPA의 어떤 장점도 적용되지 않는다. `JdbcPagingItemReader<ProductAggregateDto>` + `RowMapper`가 투영 쿼리의 **네이티브 표현 방식**이다. JPA로 짜는 것은 익숙함 외의 이득이 없다.

---

## 대안 — JdbcBatchItemWriter + ON DUPLICATE KEY UPDATE

### 코드

Writer를 JDBC 기반으로 바꾸면 이렇다.

```kotlin
@Bean
fun productRankWeeklyWriter(dataSource: DataSource): JdbcBatchItemWriter<ProductRankWeeklyRow> =
    JdbcBatchItemWriterBuilder<ProductRankWeeklyRow>()
        .dataSource(dataSource)
        .sql("""
            INSERT INTO mv_product_rank_weekly
                (year_week, product_id, `rank`, score,
                 view_count, like_count, sales_count)
            VALUES
                (:yearWeek, :productId, :rank, :score,
                 :viewCount, :likeCount, :salesCount)
            ON DUPLICATE KEY UPDATE
                `rank`      = VALUES(`rank`),
                score       = VALUES(score),
                view_count  = VALUES(view_count),
                like_count  = VALUES(like_count),
                sales_count = VALUES(sales_count)
        """.trimIndent())
        .beanMapped()
        .build()
```

JDBC URL에는 옵션 하나를 반드시 추가한다.

```
jdbc:mysql://.../commerce?rewriteBatchedStatements=true
```

이 옵션이 없으면 `addBatch()`로 쌓은 100건이 개별 INSERT 100번으로 나가고, 옵션이 있으면 **단일 멀티-밸류 INSERT 한 줄**로 재작성되어 서버 파싱 비용까지 절감된다.

### 왕복 수 비교

같은 chunk 100건 upsert 기준으로 정리하면 이렇다.

| 구분 | JpaItemWriter | JdbcBatchItemWriter |
|---|---|---|
| 존재 여부 판단 | SELECT × 100 (batching 불가) | MySQL 인덱스 lookup (서버 내부) |
| 쓰기 | INSERT/UPDATE × 100 (batch 묶음 가능) | 멀티-밸류 INSERT × 1 |
| 최소 왕복 | ~101회 | **1회** |
| 최대 왕복 | ~200회 | 1회 |

latency 1ms 기준, JpaItemWriter가 chunk당 ~200ms, JdbcBatchItemWriter가 ~1ms. chunk 2000개로 환산하면 수 분 vs 수 초 차이. 도입부의 10분 vs 30초가 여기서 온다.

---

## 오해 — ON DUPLICATE KEY UPDATE는 에러 기반이 아니다

여기서 흔한 오해 하나를 짚고 가야 한다.

> **"INSERT 해보고 중복 key 에러가 나면 UPDATE로 바꾸는 거 아닌가?"**

그렇다면 99건이 이미 존재하는 상황에서 99번의 예외 복구 비용이 발생할 것이다. 이는 어떤 batching 이득도 상쇄하고 남을 만큼 비싸다. 실제로는 그렇지 않다.

```
MySQL 내부 동작:
1. INSERT 요청 수신
2. PK 또는 UNIQUE 인덱스를 lookup
3. 충돌 예측 → 있으면 UPDATE 경로로 "내부 분기"
4. 애플리케이션 레벨 예외 발생 없음
```

MySQL은 INSERT를 "시도"한 뒤 실패를 잡는 게 아니다. **인덱스 레벨에서 충돌을 사전 예측하고 내부 경로를 바꾼다.** 예외가 없으므로 batch 안에서도 자유롭게 묶일 수 있다. 이게 `ON DUPLICATE KEY UPDATE`가 upsert의 사실상 표준이 된 이유다.

---

## 반박 방어 — "프로젝트가 JPA 기반인데 배치만 JDBC?"

PR을 올리면 반드시 나오는 반론이다.

> **팀원 A (시니어, 일관성 파):**
> "프로젝트는 JPA로 통일되어 있다. 배치만 JDBC로 가면 학습 비용이 커지고, 새벽에 돌아가는 배치 200ms 차이는 과잉 최적화 아닌가?"
>
> **팀원 B (주니어, 아키텍처 파):**
> "우리는 `domain/Repository ← infrastructure/RepositoryImpl` 구조로 DIP를 지킨다. 배치가 `JdbcTemplate`을 직접 쓰면 DIP 위반 아닌가?"

각각에 답해보자.

### "일관성"의 대상을 재정의하기

"프로젝트는 JPA로 통일되어 있다"는 말은 맞다. 그런데 **일관성의 대상이 무엇인가?**

일관성은 두 가지로 나뉜다.

- **도구 일관성** — 모두 JPA를 쓴다
- **아키텍처 일관성** — 레이어드 구조, 에러 처리(CoreException), API 응답(ApiResponse), 테스트 패턴(@Nested + @DisplayName), ArchUnit 아키텍처 테스트

지켜야 할 것은 **아키텍처 일관성**이다. 도구 일관성은 목적이 아니라 결과다. 같은 문제에 같은 도구가 맞기 때문에 자연스레 일관돼 보이는 것이지, 일관성 자체를 위해 잘못된 도구를 쓰는 것은 본말전도다.

배치 모듈이 JDBC로 짜여도:
- 레이어드 구조는 그대로
- 에러 처리 규약도 그대로
- 테스트 패턴도 그대로
- ArchUnit 규칙도 그대로 통과

**"JPA냐 JDBC냐"는 기술 선택이고, 아키텍처 일관성과 독립된 축이다.**

### "과잉 최적화" 반박 — 새벽 200ms가 아니다

"200ms 차이"라는 프레이밍이 함정이다. 실제로는 이렇다.

- **일상 운영**: 100만 상품 기준 10분 vs 30초. 다른 새벽 배치와의 스케줄 충돌, 재시도 창 확보 차이
- **장애 복구**: 30분 배치가 실패하면 업무시간 재실행 시 DB 락 경합 발생. 2~3분 배치는 영향 없이 재실행 가능
- **가중치 변경 재실행**: "가중치 바꿨으니 지난주 다시 돌려봐"가 10초에 끝나야 "일단 돌려보자"가 가능. 10분이면 기각된다

**배치가 빠르다는 것은 단순히 "빠르다"가 아니라, 재실행 자유도라는 운영 자산이다.** 이게 없으면 "배치는 건드리지 않는 것이 상책"이라는 조직 문화가 생기고, 데이터 가설 검증이 느려진다.

### DIP 방어 — abstraction 개수 경쟁이 아니다

DIP를 "모든 외부 호출에 인터페이스를 끼우기"로 이해하면 답이 꼬인다. DIP의 본질은 **"도메인이 인프라 세부에 의존하지 않는다"**이고, 판단 기준은 호출 방향이다.

- `domain/Repository` 인터페이스: **도메인 Service가 호출한다** → 인프라 세부를 도메인에서 숨기기 위한 추상화
- Spring Batch의 `ItemReader<T>` / `ItemWriter<T>`: **Spring Batch 프레임워크가 호출한다** (`StepBuilder`의 chunk 루프) → 프레임워크 확장점

배치 Reader를 호출하는 것은 도메인 Service가 아니다. 도메인은 이 Reader의 존재 자체를 모른다. 따라서 **DIP의 적용 대상이 아니다.** 여기에 억지로 `domain/Repository` 인터페이스를 하나 더 얹으면 2중 추상화가 되고, Spring Batch 프레임워크의 restart/skip/retry 기능을 놓치게 된다.

```
apps/commerce-batch/
  batch/weekly-ranking/
    WeeklyRankingJobConfig.kt      # Job/Step 정의
    ProductAggregateReader.kt      # JdbcPagingItemReader Bean
    WeightScoreProcessor.kt        # 가중치 Java 계산
    ProductRankWeeklyWriter.kt     # JdbcBatchItemWriter Bean
```

배치 모듈은 이 구조로 완결된다. `domain/Repository`와 완전 분리되며, Spring Batch 네이티브 인터페이스를 활용한다. DIP 위반이 아니라 **DIP의 적용 범위 밖**이다.

---

## 마무리 — JPA의 장점을 전부 꺼야 성능이 나온다면

JPA를 배치에서 "잘 쓰려면" 이런 과정을 거친다.

- Reader에 `@QueryHints(HINT_READONLY=true)`로 Dirty Checking을 끈다
- `hibernate.jdbc.batch_size`, `order_inserts`, `order_updates`를 튜닝한다
- Entity 대신 `SELECT new Dto(...)` 생성자 투영으로 집계를 표현한다
- `transacted`, `pageSize`, `chunkSize` 관계를 점검해 영속성 컨텍스트 수명을 제어한다
- merge 대신 native `INSERT ... ON DUPLICATE KEY UPDATE`를 JPQL이나 `@Modifying`으로 우회한다

이 조치들이 끝나면 남는 질문은 하나다.

> **이 시점에서 JPA를 쓰는 이유가 무엇인가?**

영속성 컨텍스트의 자동 관리, 연관관계 탐색, Dirty Checking, Entity 상태 머신 — JPA가 **"대신 해줘서 편했던 모든 것"**을 끈 채로 쓰고 있다. 남는 건 "JpaRepository를 주입받는 익숙한 모양" 정도다. 그 익숙함을 위해 20배의 성능을 포기하는 것은 합리적 교환이 아니다.

> **도구의 장점을 전부 꺼야 성능이 나온다면, 도구 선택이 잘못된 것이다.**

JPA는 도메인 Entity를 주고받는 트랜잭션 경계에서 빛난다. 상품 주문이 들어오면 `Order`, `Stock`, `Payment` Entity가 함께 움직이고, 각 필드가 바뀌면 Dirty Checking이 UPDATE를 만들어준다. 영속성 컨텍스트가 중복 조회를 막아준다. 이 영역에서 JPA는 대체 불가능하다.

하지만 배치는 다르다. 배치는 **"raw 데이터를 대량으로 훑어 집계 DTO를 쏟아붓는"** 작업이다. Entity 생명주기도, 연관관계도, 변경 감지도 필요 없다. 그 일에는 그 일에 맞는 도구가 있다. `JdbcPagingItemReader`와 `JdbcBatchItemWriter`, 그리고 `ON DUPLICATE KEY UPDATE` 한 줄이다.

**"배치만 JDBC로 내려가는 것"은 일관성의 훼손이 아니라 적재적소다.** 도메인 모듈의 JPA와 배치 모듈의 JDBC는 충돌하지 않는다. 둘 다 아키텍처 규약(레이어드, 에러 처리, 테스트 패턴)을 지킨다. 그것이 진짜 일관성이다.

JPA를 버리는 것이 아니라, **JPA가 있을 자리와 없어야 할 자리를 구분하는 것.** 배치에서 JDBC를 고르는 결정의 본질은 거기에 있다.
