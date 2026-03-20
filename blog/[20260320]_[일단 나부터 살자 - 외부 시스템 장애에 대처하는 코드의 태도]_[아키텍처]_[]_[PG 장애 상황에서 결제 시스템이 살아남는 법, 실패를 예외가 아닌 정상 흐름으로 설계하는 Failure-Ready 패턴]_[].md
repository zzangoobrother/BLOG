# 일단 나부터 살자 — 외부 시스템 장애에 대처하는 코드의 태도

## 들어가며

PG사 서버가 죽으면 우리 결제 시스템도 죽어야 하는 걸까?

당연히 아니다. 그런데 많은 코드가 그렇게 짜여 있다. 외부 API를 호출하고, 응답이 안 오면 예외를 던지고, 그 예외가 Controller까지 올라가서 500 에러를 뱉는다. PG가 죽었을 뿐인데, 우리 서비스 전체가 흔들린다.

외부 시스템을 호출하면서 많이 고민한 건 기술 뿐만 아니라 **"외부 시스템이 실패했을 때, 우리 코드는 어떤 태도를 취해야 하는가?"**라는 질문이었다.

결론부터 말하면, **실패를 예외로 보지 않고 정상 흐름의 일부로 받아들이는 것**이 핵심이었다. 이 글에서는 그 판단의 과정을 정리해보려고 한다.

1. 실패를 예외로 다루면 벌어지는 일
2. 태도를 바꾸면 — 실패도 정상 흐름이다
3. 이 태도가 코드에 어떻게 드러나는가
4. 나부터 살고, 그 다음에 확인한다

---

## 실패를 예외로 다루면 벌어지는 일

외부 API 호출 코드를 처음 짜면 보통 이렇게 된다.

```kotlin
fun requestPayment(orderId: String, amount: Long): PaymentResponse {
    val response = pgClient.requestPayment(orderId, amount) // PG 호출
    return PaymentResponse(response.transactionKey, response.status)
}
```

문제없어 보인다. PG가 응답하면 잘 동작한다.

그런데 PG가 응답하지 않으면? 타임아웃이 나면? 500을 뱉으면?

```
Client → 우리 서버 → PG 서버 (장애)
                        ↓
                   ReadTimeoutException
                        ↓
                   500 Internal Server Error
                        ↓
              "결제에 실패했습니다" (하지만 PG에서는 결제가 됐을 수도 있다)
```

여기서 진짜 문제는 500 에러가 아니다. **결제가 됐는지 안 됐는지 모른다는 것**이 문제다. 타임아웃은 "PG가 처리를 안 했다"가 아니라 "응답을 못 받았다"는 뜻이다. PG는 결제를 완료했는데 우리만 모르는 상황이 생길 수 있다.

그리고 또 하나. PG가 1초만 느려져도 우리 서버의 스레드가 묶인다. 200개의 Tomcat 스레드가 전부 PG 응답을 기다리고 있으면, 결제와 상관없는 상품 조회, 로그인 같은 API도 전부 멈춘다. **남의 장애가 내 장애가 된다.**

```
[PG 1초 지연 시]

Tomcat 스레드 200개 중:
  → 결제 API 대기: 180개 (PG 응답 대기)
  → 상품 조회:      처리 불가 (스레드 부족)
  → 로그인:          처리 불가 (스레드 부족)
  → 주문 목록:       처리 불가 (스레드 부족)

결제만 느린 건데, 서비스 전체가 죽는다.
```

정리하면 두 가지 문제다.

1. **상태를 모른다** — 실패한 건지, 성공했는데 응답만 못 받은 건지
2. **전파된다** — PG 장애가 우리 서비스 전체로 번진다

---

## 태도를 바꾸면 — 실패도 정상 흐름이다

문제를 다시 보자. 근본적인 원인은 "PG가 반드시 응답할 것"이라는 가정이다. 외부 시스템은 언제든 실패할 수 있다. 이건 예외 상황이 아니라 **당연한 현실**이다.

그래서 태도를 바꿨다.

> **PG가 응답하면 좋고, 응답하지 않아도 우리는 할 일을 한다.**

이 태도를 코드로 표현하면 이렇게 된다.

```kotlin
// 기존: PG가 실패하면 예외가 터진다
fun requestPayment(...): PaymentResponse {
    return pgClient.requestPayment(...)  // 여기서 예외 → 전파
}

// 변경: PG가 실패해도 null이 올 뿐이다
fun requestPayment(...): PaymentGatewayResponse? {
    return try {
        pgClient.requestPayment(...)
    } catch (e: Exception) {
        null  // 실패 = null, 예외를 삼킨다
    }
}
```

`null`이다. 예외가 아니다.

이 차이가 작아 보이지만, 코드 전체의 흐름을 바꾼다. 예외가 터지면 "어떻게 복구하지?"를 고민해야 하지만, null이 오면 "그래서 다음 단계는 뭐지?"를 고민하게 된다. **실패가 프로그램의 중단이 아니라 분기 조건이 되는 것**이다.

---

## 이 태도가 코드에 어떻게 드러나는가

### 도메인 인터페이스 — "응답이 없을 수 있다"를 타입으로 선언한다

```kotlin
interface PaymentGateway {
    fun requestPayment(
        userId: String,
        orderId: String,
        cardType: String,
        cardNo: String,
        amount: Long,
        callbackUrl: String,
    ): PaymentGatewayResponse?  // ← nullable. PG가 응답 못 할 수 있다.

    fun getTransactionStatus(
        userId: String,
        transactionKey: String,
    ): PaymentGatewayTransactionDetail?  // ← 이것도 nullable

    fun getTransactionsByOrderId(
        userId: String,
        orderId: String,
    ): List<PaymentGatewayResponse>  // ← 빈 리스트로 표현
}
```

이 인터페이스는 도메인 레이어에 있다. JPA도 Spring도 모른다. 그런데 반환 타입이 전부 nullable이거나 빈 리스트다. **"PG가 응답하지 않을 수 있다"는 사실이 타입 시스템에 녹아 있다.**

예외를 던지는 인터페이스는 "반드시 성공해야 한다"는 계약이다. nullable을 반환하는 인터페이스는 "실패할 수 있고, 그건 정상이다"라는 계약이다. 같은 인터페이스지만 태도가 완전히 다르다.

### 인프라 구현체 — 모든 실패를 null로 변환한다

```kotlin
@Component
class PaymentGatewayImpl(
    private val pgClient: PgClient,
) : PaymentGateway {

    @Bulkhead(name = "pg-simulator")
    override fun requestPayment(
        userId: String,
        orderId: String,
        cardType: String,
        cardNo: String,
        amount: Long,
        callbackUrl: String,
    ): PaymentGatewayResponse? {
        return try {
            val data = pgClient.requestPayment(
                PgPaymentRequest(userId, orderId, cardType, cardNo, amount, callbackUrl)
            ).data ?: return null

            PaymentGatewayResponse(
                transactionKey = data.transactionKey,
                status = data.status,
                reason = data.reason,
            )
        } catch (e: Exception) {
            log.warn("PG 결제 요청 실패 (orderId={}): {}", orderId, e.message)
            null
        }
    }
}
```

`catch (e: Exception) → null`. 타임아웃이든, CircuitBreaker가 열렸든, PG가 500을 뱉었든, 결과는 같다. **"PG가 응답하지 못했다"는 하나의 사실만 상위 레이어에 전달한다.**

로그는 남긴다. 하지만 예외를 전파하지 않는다. infrastructure 레이어에서 예외를 삼키고, 도메인에는 null이라는 값만 전달한다. **계층 간 책임이 분리되는 것**이다.

여기서 한 가지 주의할 점이 있다. `catch (e: Exception)`은 모든 예외를 삼킨다. PG 타임아웃뿐 아니라 우리 코드의 버그(NullPointerException, ClassCastException 등)까지도. 엄밀하게는 외부 시스템 장애에 해당하는 예외만 잡는 것이 맞다. 이 프로젝트에서는 FeignClient + Fallback 조합이 외부 장애를 `PgUnavailableException`이라는 하나의 예외로 변환해주고, 그 외의 예외가 발생할 여지가 구조적으로 적기 때문에 이렇게 처리했다. 만약 매핑 로직이 복잡해진다면, 잡는 예외 타입을 좁히는 것을 고려해야 한다.

### Facade — null이 오면 한 번 더 확인하고, 그래도 안 되면 기록한다

여기가 핵심이다.

```kotlin
@Transactional
fun requestPayment(
    userId: Long,
    orderId: String,
    cardType: CardType,
    cardNo: String,
    amount: Long,
): PaymentInfo {
    // 1. 결제를 일단 REQUESTED 상태로 저장한다 — 무슨 일이 있어도 기록은 남긴다
    val payment = paymentService.createPayment(userId, orderId, cardType, cardNo, amount)

    // 2. PG를 호출한다 — 실패하면 null
    val pgResponse = paymentGateway.requestPayment(
        userId = userId.toString(),
        orderId = orderId,
        cardType = cardType.name,
        cardNo = cardNo,
        amount = amount,
        callbackUrl = callbackUrl,
    )

    // 3. null이면 한 번 더 확인한다 — 혹시 PG에서는 처리했는데 응답만 못 받은 건 아닌지
    val transactionKey = pgResponse?.transactionKey
        ?: paymentGateway.getTransactionsByOrderId(userId.toString(), orderId)
            .firstOrNull()?.transactionKey

    // 4. 그래도 없으면 실패로 기록한다 — 하지만 결제 기록 자체는 남아있다
    if (transactionKey != null) {
        paymentService.markPending(payment.id, transactionKey)
    } else {
        paymentService.markFailed(payment.id, "PG 결제 요청에 실패했습니다. 다시 시도해주세요.")
    }

    return PaymentInfo.from(paymentService.getPayment(payment.id))
}
```

이 코드에는 try-catch가 없다. 예외가 올 수 없기 때문이다. `PaymentGateway`가 이미 모든 예외를 null로 변환했으니까.

흐름을 다시 보자.

```
1. 결제 기록 생성 (REQUESTED)
     ↓
2. PG 호출 → 성공? → transactionKey 획득 → PENDING
     ↓ (null)
3. PG에 주문 ID로 재조회 → 있었다? → transactionKey 획득 → PENDING
     ↓ (null)
4. 정말 없다 → FAILED로 기록
```

**어떤 경우에도 결제 기록은 DB에 남는다.** REQUESTED, PENDING, FAILED 중 하나의 상태로. PG가 죽어도, 타임아웃이 나도, 우리 시스템은 자신의 상태를 정확히 알고 있다.

3번 단계가 특히 중요하다. 타임아웃이 났다는 건 "PG가 처리를 안 했다"가 아니다. "우리가 응답을 못 받았다"일 뿐이다. 그래서 한 번 더 물어본다. "혹시 처리한 거 아니야?" 이 한 번의 재확인이 **결제는 됐는데 우리만 모르는 유령 결제**를 방지한다.

---

## 남의 장애가 나한테 번지지 않게 — 격벽과 차단기

태도를 바꾸는 것만으로는 부족하다. PG가 느려지면 스레드가 묶이는 문제가 남아있다. 여기에는 두 가지 장치가 필요하다.

### 격벽(Bulkhead) — PG 호출에 쓸 수 있는 스레드를 제한한다

```yaml
resilience4j:
  bulkhead:
    instances:
      pg-simulator:
        max-concurrent-calls: 25    # PG 동시 요청 최대 25개
        max-wait-duration: 0        # 대기 없이 즉시 거부
```

Tomcat 스레드가 200개라면, PG 호출에는 최대 25개만 쓸 수 있다. 나머지 175개는 상품 조회, 로그인 같은 다른 API를 처리한다.

```
[Bulkhead 적용 후]

Tomcat 스레드 200개 중:
  → 결제 API:    최대 25개 (PG 대기)
  → 상품 조회:    정상 처리 (175개 사용 가능)
  → 로그인:       정상 처리
  → 주문 목록:    정상 처리

PG가 느려져도, 결제 외의 기능은 멀쩡하다.
```

**배가 침몰할 때 격벽이 물을 한 구역에 가두는 것**과 같다. PG 장애라는 물이 들어와도, 그 물은 결제라는 한 구역에만 머문다. 나머지 구역은 안전하다.

### 차단기(CircuitBreaker) — PG가 계속 실패하면 호출 자체를 멈춘다

```yaml
resilience4j:
  circuitbreaker:
    instances:
      pg-simulator:
        sliding-window-size: 10            # 최근 10건 기준
        failure-rate-threshold: 50         # 50% 이상 실패하면
        wait-duration-in-open-state: 10s   # 10초간 호출 중단
        permitted-number-of-calls-in-half-open-state: 3  # 3건만 시험 삼아 보내본다
```

PG가 계속 실패하는데 요청을 계속 보내는 건 의미가 없다. 오히려 죽어가는 PG에 부하를 더 주는 꼴이다. 차단기는 실패율이 임계치를 넘으면 호출 자체를 차단한다.

```
[CircuitBreaker 상태 전이]

CLOSED (정상)
  → 최근 10건 중 5건 이상 실패
  → OPEN (차단 — PG 호출 안 함)
      → 10초 대기
      → HALF_OPEN (3건만 시험)
          → 성공 → CLOSED (복구)
          → 실패 → OPEN (다시 차단)
```

차단기가 열리면 PG를 호출하지 않고 즉시 Fallback으로 빠진다. Fallback에서 예외를 던지고, `PaymentGatewayImpl`이 그 예외를 잡아서 null을 반환한다. 위에서 설계한 흐름 그대로 동작한다. **차단기가 열려도 결제 기록은 FAILED로 정상 저장된다.**

---

## 재시도에도 태도가 있다 — 멱등한 것만 재시도한다

PG 호출이 실패했을 때, 재시도를 하면 안 되는 경우가 있다.

```kotlin
// 결제 요청 — 재시도하면 안 된다
@Bulkhead(name = "pg-simulator")
override fun requestPayment(...): PaymentGatewayResponse? {
    // @Retry 없음. 2번 호출하면 2건 결제될 수 있다.
}

// 결제 상태 조회 — 재시도해도 된다
@Bulkhead(name = "pg-simulator")
@Retry(name = "pg-query")
override fun getTransactionStatus(...): PaymentGatewayTransactionDetail? {
    // @Retry 있음. 조회는 몇 번을 해도 결과가 같다.
}
```

결제 요청은 **비멱등**하다. 한 번 보내면 한 건 결제, 두 번 보내면 두 건 결제. 재시도하면 이중 결제가 발생한다.

결제 상태 조회는 **멱등**하다. 열 번을 물어봐도 결과는 같다. 이건 재시도해도 된다.

```yaml
resilience4j:
  retry:
    instances:
      pg-query:                            # 조회 전용 재시도 정책
        max-attempts: 3                    # 최대 3회
        wait-duration: 500ms               # 500ms → 1s → 2s (지수 백오프)
        enable-exponential-backoff: true
        retry-exceptions:
          - java.io.IOException
          - feign.RetryableException
```

이 구분을 테스트로도 검증한다.

```kotlin
@Test
fun `결제 요청에는 retry가 적용되지 않아야 한다`() {
    val retryOptional = retryRegistry.find("pg-simulator")
    assertThat(retryOptional).isEmpty  // pg-simulator에는 retry 없음
}

@Test
fun `결제 조회에는 retry가 적용되어야 한다`() {
    val retryOptional = retryRegistry.find("pg-query")
    assertThat(retryOptional).isPresent  // pg-query에는 retry 있음
}
```

**"재시도해도 되는가?"는 기술의 문제가 아니라 비즈니스의 문제다.** Resilience4j가 retry를 지원하니까 다 걸어놓는 게 아니라, "이 동작을 두 번 해도 괜찮은가?"를 먼저 물어야 한다.

---

## 콜백도 믿지 않는다

PG가 결제 결과를 콜백으로 알려준다. 그런데 이 콜백을 그대로 믿어도 될까?

```kotlin
@Transactional
fun handleCallback(transactionKey: String, status: String, reason: String?) {
    val payment = paymentService.getPaymentByTransactionKey(transactionKey)
    val orderId = payment.orderId

    // 콜백 데이터를 그대로 쓰지 않는다 — PG에 직접 확인한다
    val pgDetail = paymentGateway.getTransactionStatus(
        payment.userId.toString(), transactionKey
    )

    val verifiedStatus = pgDetail?.status ?: status   // PG 조회 우선, 불가능하면 콜백
    val verifiedReason = pgDetail?.reason ?: reason

    when (verifiedStatus) {
        "SUCCESS" -> {
            paymentService.markSuccess(payment.id)
            orderService.changeStatus(orderId, OrderStatus.CONFIRMED)
        }
        "FAILED" -> {
            paymentService.markFailed(payment.id, verifiedReason ?: "결제 실패")
            cancelOrderWithCompensation(orderId)  // 재고, 쿠폰 복구
        }
    }
}
```

콜백이 "성공"이라고 해도, PG에 직접 물어봐서 확인한다. 콜백은 위변조될 수 있고, 네트워크 문제로 잘못된 데이터가 올 수도 있다. **외부에서 오는 데이터는 신뢰하지 않고 검증한다.**

그리고 PG 조회마저 실패하면(`pgDetail`이 null이면)? 그때는 콜백 데이터를 쓴다. 완벽하지는 않지만, 아무것도 안 하는 것보다 낫다. **최선을 추구하되, 차선도 준비하는 것**이다.

---

## 전체 흐름 정리

```
[정상 흐름]
결제 요청 → PG 응답 → transactionKey 획득 → PENDING → 콜백 → PG 재확인 → SUCCESS

[PG 응답 실패, 하지만 처리는 됐을 때]
결제 요청 → PG 타임아웃(null) → PG 재조회 → transactionKey 발견 → PENDING → 이후 동일

[PG 완전 장애]
결제 요청 → PG 타임아웃(null) → PG 재조회(빈 리스트) → FAILED → 기록 남김 → 재시도 가능

[CircuitBreaker OPEN]
결제 요청 → 호출 차단(null) → FAILED → 기록 남김 → PG 복구 후 재시도 가능
```

어떤 시나리오에서도 **결제 기록은 남고, 시스템은 살아있고, 다른 기능은 영향받지 않는다.**

---

## 마무리 — 일단 나부터 살자

외부 시스템에 의존하는 코드를 짤 때, 기술보다 먼저 정해야 하는 건 **태도**다.

"PG가 응답할 것이다"라는 낙관 위에 세운 코드는, PG가 1초만 느려져도 무너진다. "PG가 응답하지 않을 수 있다"는 현실 위에 세운 코드는, PG가 아예 죽어도 자기 할 일을 한다.

이 태도가 코드에 드러나는 방식을 정리하면 이렇다.

> **실패를 예외가 아닌 값으로 표현한다**
> - nullable 반환으로 "응답이 없을 수 있다"를 타입에 녹인다
> - 호출자는 try-catch가 아니라 null 체크로 분기한다
> - 예외가 계층을 관통하지 않는다

> **실패해도 상태를 기록한다**
> - PG 호출 전에 REQUESTED로 저장한다
> - 성공이든 실패든 최종 상태를 DB에 반영한다
> - 유령 결제나 유실 결제가 발생하지 않는다

> **남의 장애가 나한테 번지지 않게 막는다**
> - 격벽(Bulkhead)으로 PG 호출 스레드를 제한한다
> - 차단기(CircuitBreaker)로 반복 실패 시 호출을 중단한다
> - PG가 죽어도 상품 조회, 로그인은 정상 동작한다

> **재시도에도 기준이 있다**
> - 멱등한 조회만 재시도한다
> - 비멱등한 결제 요청은 재시도하지 않는다
> - "해도 되는가?"를 먼저 묻고, 그 다음에 기술을 적용한다

외부 시스템은 언제든 실패한다. 이건 예외적인 상황이 아니라 일상이다. 우리가 통제할 수 없는 것에 의존할 때, 가장 먼저 해야 할 일은 **"그게 안 되면 나는 어떻게 할 것인가"를 정하는 것**이다.

기술은 그 다음이다. CircuitBreaker든 Retry든 Bulkhead든, 그 밑에 깔려 있는 건 결국 하나의 원칙이다.

**일단 나부터 살자.**
