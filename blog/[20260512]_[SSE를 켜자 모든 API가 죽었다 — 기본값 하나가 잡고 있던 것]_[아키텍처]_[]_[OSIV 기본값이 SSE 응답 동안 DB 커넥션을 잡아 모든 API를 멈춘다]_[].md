# SSE를 켜자 모든 API가 죽었다 — 기본값 하나가 잡고 있던 것

## 들어가며

실시간 예약 현황 알림을 SSE로 띄웠다. 단순한 기능이다. 구현도 간단하다. 클라이언트가 연결하면 예약 변경이 생길 때마다 이벤트를 밀어준다.

잘 돌아갔다. 테스트도 통과했다.

부하 테스트 중, HikariCP 로그에 처음 보는 메시지가 찍혔다. 그 순간부터 **예약 API만 죽은 게 아니었다. 로그인, 결제, 모든 API가 줄줄이 timeout으로 쌓였다.**

우리는 DB 호출을 늘린 적이 없다. 트랜잭션을 길게 잡은 적도 없다. SSE 핸들러는 DB를 직접 건드리지도 않는다. 그런데 왜?

이 글에서는 세 가지 질문을 순서대로 답한다.

1. SSE 엔드포인트가 DB를 직접 잡지 않는데 왜 커넥션이 마르는가
2. SSE 한 기능의 문제가 왜 시스템 전체로 번지는가
3. 결국 무엇이 답이고, 그 답이 왜 "한 줄"로 끝나는가

---

## 1. 첫 가설 — 의심은 SSE 엔드포인트로 향했다

"SSE 핸들러에서 뭔가 잘못한 거다." 코드를 뒤졌다.

```kotlin
@GetMapping("/reservations/stream", produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
fun streamReservations(): SseEmitter {
    val emitter = SseEmitter(Long.MAX_VALUE)
    sseEmitterRegistry.register(emitter)
    return emitter
}
```

`SseEmitter`를 생성하고 반환한다. 직접 DB를 호출하는 코드는 없다. `@Transactional`도 없다. 이 메서드가 DB 커넥션을 잡을 이유가 없다.

가설이 깨졌다. 다른 곳을 봐야 한다.

---

## 2. 로그가 가리킨 것 — HikariCP의 신호

HikariCP 로그에는 이게 찍혀 있었다.

```log
HikariPool-1 - Connection is not available, request timed out after 30000ms
```

그리고 메트릭을 열어봤더니 이상한 장면이 있었다.

```
Active connections : 10  (풀 max)
Idle connections  :  0
Pending threads   : 37
```

Active connection이 풀 max인 10에 고정된 채 **줄어들지 않았다.** 새 요청이 들어올수록 Pending만 쌓였다.

SSE 동시 연결 수를 세어봤다. 10명이었다. 잡혀 있는 connection 수와 정확히 일치했다.

즉, **SSE 연결당 커넥션 하나씩 누군가가 잡고 놓지 않고 있다.** SSE 연결이 살아 있는 내내.

그런데 SSE 핸들러는 DB를 안 잡는다. 그럼 누가 잡는가?

---

## 3. 범인 — Spring Boot가 기본으로 켜둔 OSIV

Spring Boot의 기본값을 확인했다.

```yaml
spring:
  jpa:
    open-in-view: true  # Spring Boot 기본값
```

`spring.jpa.open-in-view=true`. **OSIV(Open Session In View)**다.

이 설정이 켜져 있으면 `OpenEntityManagerInViewInterceptor`가 동작한다. 메커니즘은 이렇다.

1. HTTP 요청이 들어온다 → Interceptor가 EntityManager를 생성한다
2. EntityManager가 DB connection을 획득한다
3. 컨트롤러 로직이 실행된다
4. **응답이 완료될 때까지 connection을 유지한다** (뷰/컨트롤러에서 lazy loading 가능하도록)
5. 응답이 끝나면 connection을 반환한다

일반 REST API에서는 이 메커니즘이 거의 안 보인다. 응답이 50ms 안에 끝나기 때문이다. connection을 잠깐 잡았다가 금방 돌려준다.

SSE는 다르다. **SSE는 응답이 끝나지 않는다.** 클라이언트가 연결을 끊을 때까지 응답 스트림은 열려 있다. 분, 시간 단위로.

OSIV는 그동안 connection을 반환하지 않는다. SSE가 "응답 중"이니까.

SSE 핸들러가 DB를 직접 쓰지 않아도, OSIV가 응답 시작 시점에 이미 connection을 잡아버린 것이다.

---

## 4. 왜 평소엔 안 보였나 — 점유 시간이 만드는 격차

산수로 비교하면 명확하다.

| 상황 | 응답 시간 | 동시 요청 | 풀 max | 결과 |
|---|---|---|---|---|
| 일반 REST API | 50ms | 1,000명 | 10 | 풀이 빠르게 회전. 충분 |
| SSE | 10분 | 11명 | 10 | 풀 고갈. 나머지 989명 차단 |

같은 OSIV 설정이다. 같은 풀 크기다. 다른 결과를 만든 건 단 하나, **점유 시간**이다.

일반 API는 connection을 50ms 동안 잡는다. 10개 풀로 초당 200명을 처리할 수 있다. 풀이 빠르게 돈다.

SSE는 connection을 10분 동안 잡는다. 11명째 연결이 들어오는 순간 풀이 마른다. 그리고 **그 순간 로그인·결제·예약 조회, 시스템의 모든 API가 connection을 기다리며 멈춘다.**

이것이 장애 반경(blast radius)이 커지는 메커니즘이다. 한 기능의 점유가 길어지면, 공유 자원을 쓰는 전체 시스템이 영향을 받는다.

SSE가 12명째, 50명째로 늘어날수록 이 반경은 더 빠르게 확대된다.

---

## 5. 해결 — 한 줄, 그러나 트레이드오프

해결은 간단하다.

```yaml
spring:
  jpa:
    open-in-view: false
```

이 한 줄을 추가하면 OSIV가 꺼진다. Interceptor가 동작하지 않는다. SSE 응답 동안 DB connection을 잡지 않는다.

하지만 끄면 사이드 이펙트가 있다. OSIV가 열어둔 EntityManager 범위 밖에서 lazy loading을 시도하면 `LazyInitializationException`이 발생한다.

예를 들어, OSIV가 켜져 있을 때는 이런 코드가 돌아갔다.

```kotlin
// OSIV 켜진 상태 — 컨트롤러에서 lazy 필드 접근이 가능
@GetMapping("/reservations/{id}")
fun getReservation(@PathVariable id: Long): Reservation {
    val reservation = reservationService.findById(id)
    reservation.venue.name  // 여기서 lazy loading 발생 — OSIV가 connection을 열어둬서 가능
    return reservation
}
```

OSIV를 끄면 이 코드는 `LazyInitializationException`을 던진다. 트랜잭션 바깥에서 lazy 필드에 접근하기 때문이다.

해결 방향은 하나다. 트랜잭션 경계 안, 즉 서비스 레이어에서 DTO 변환을 끝내야 한다.

```kotlin
// OSIV 끈 상태 — 서비스에서 DTO 변환 완료 후 반환
@Transactional(readOnly = true)
fun findReservationDetail(id: Long): ReservationDetailDto {
    val reservation = reservationRepository.findById(id)
        ?: throw CoreException(ErrorType.NOT_FOUND)
    return ReservationDetailDto(
        id = reservation.id,
        venueName = reservation.venue.name,  // 트랜잭션 안에서 접근 — 정상
        startAt = reservation.startAt
    )
}
```

이 규율은 사실 "처음부터 그래야 했던" 규율이다. OSIV는 그 규율을 *생략할 수 있게* 해주는 편의 기능이었다. 그 편의가 long-lived 응답에서 비용으로 돌아온 것이다.

---

## 6. 본질 — 기본값은 가정을 깐다

OSIV 기본값 ON은 **짧은 동기 응답**을 가정한 세계에서 만들어졌다. 요청이 들어오고, 서비스 로직이 실행되고, 응답이 나가고, connection이 반환된다. 이 흐름이 ms 단위로 끝난다는 가정 위에서 OSIV는 안전하게 작동한다.

그 가정이 깨지는 순간들이 있다.

- **SSE**: 응답이 분 단위로 길어진다. 클라이언트가 연결을 유지하는 한 응답은 끝나지 않는다.
- **WebSocket**: 응답이 시간 단위로 길어진다.
- **스트리밍 다운로드**: 큰 파일을 전송하는 동안 응답이 열려 있다.

이 기술들의 공통점은 **응답 시간 모델이 다르다**는 것이다. HTTP 요청/응답의 짧은 사이클이 아니라, 연결이 살아 있는 내내 응답 스트림이 유지된다.

새 기술을 도입할 때 물어야 할 질문은 결국 하나다.

> **이 기술이 공유 자원의 점유 시간을 바꾸는가?**

점유 시간이 바뀌면, 그 자원을 둘러싼 모든 기본값의 의미가 바뀐다. OSIV만의 이야기가 아니다. 커넥션 풀 크기, 타임아웃 설정, 스레드 풀 크기 — 짧은 응답을 가정하고 정해진 모든 값이 다시 검토 대상이 된다.

---

## 마치며

"기본값을 의심하라"는 조언은 흔하다. 하지만 이 조언은 보통 자기 코드를 다 뒤진 뒤에야 떠오른다.

SSE 핸들러는 결백했다. DB를 호출하지 않았다. 트랜잭션도 없었다. 범인은 프레임워크가 조용히 켜둔 기본값이었다. 그 기본값이 만들어질 때 상상하지 못한 방식으로 연결을 쓰는 기술을 들이는 순간, 기본값의 의미가 바뀐다.

SSE는 좋은 기술이다. 단, 자원 점유 모델을 바꾸는 기술이다. 그 기술을 시스템에 들일 때는 기능 동작뿐 아니라 **공유 자원에 대한 어떤 가정을 흔드는가**를 함께 봐야 한다.
