# WIL — TDD, 테스트 종류, 단위 테스트, 그리고 Test Doubles

## 이번 주에 무엇을 학습했는가

1. **TDD란 무엇인가** — 정의와 Red-Green-Refactor 사이클
2. **테스트의 종류** — 단위 / 통합 / E2E, 그리고 테스트 피라미드
3. **Test Doubles** — Mock, Stub, Spy, Fake, Dummy의 차이
4. **단위 테스트의 세부 분류** — 상태 검증, 행위 검증, 경계값, 예외 검증 등

---

## 1. TDD — "테스트 기법"이라고 생각했는데, "설계 방법론"이었다

### 중점 학습 내용

TDD(Test-Driven Development)는 **테스트를 먼저 작성하고, 그 테스트를 통과시키는 코드를 구현하는 개발 방법론**이다. Kent Beck이 체계화했고, 핵심 사이클은 이렇다.

```
Red(실패하는 테스트) → Green(최소 구현으로 통과) → Refactor(코드 개선) → 반복
```

```java
// Red — 아직 Email 클래스가 없다. 컴파일조차 안 된다.
@Test
void 유효한_이메일_형식이면_생성된다() {
    Email email = Email.of("user@example.com");
    assertThat(email.getValue()).isEqualTo("user@example.com");
}

// Green — 통과만 시킨다
public static Email of(String value) {
    return new Email(value);
}

// Refactor — 검증 로직을 추가한다
public static Email of(String value) {
    if (value == null || value.isBlank()) {
        throw new IllegalArgumentException("이메일은 비어있을 수 없습니다.");
    }
    if (!EMAIL_PATTERN.matcher(value).matches()) {
        throw new IllegalArgumentException("올바른 이메일 형식이 아닙니다.");
    }
    return new Email(value);
}
```

### 헷갈렸던 것

TDD를 "테스트를 잘 작성하는 기법"이라고 생각했다. 그래서 "테스트 커버리지를 높이는 것"이 목표라고 착각했다.

그런데 실제로 해보니 **테스트를 먼저 쓴다는 건 곧 "이 코드를 사용하는 입장에서 인터페이스를 먼저 설계한다"는 뜻**이었다.

### 결국 어떻게 이해했는가

**TDD는 테스트 기법이 아니라 설계 방법론**이다. "어떻게 구현할까"보다 "어떻게 사용될까"를 먼저 고민하게 만드는 도구다.

그리고 짧은 사이클을 반복하는 것이 핵심이다.

> 테스트가 어려우면 설계를 의심해라. 의존성이 너무 많거나 한 클래스가 너무 많은 일을 하고 있을 가능성이 높다.

---

## 2. 테스트의 종류 — 피라미드의 의미를 체감하다

### 중점 학습 내용

소프트웨어 테스트는 검증 범위에 따라 세 레벨로 나뉜다.

- E2E 테스트 : 느리고, 비싸고, 하지만 실제 사용자 시나리오를 검증
- 통합 테스트 : 컴포넌트 간 연동을 검증
- 단위 테스트 : 빠르고, 저렴하고, 가장 많이 작성

**단위 테스트 (Unit Test)** — 가장 작은 단위(메서드, 클래스)를 외부 의존성 없이 검증

```java
@Test
void 비밀번호에_생년월일이_포함되면_예외가_발생한다() {
    String rawPassword = "Test19900115!@";
    LocalDate birthday = LocalDate.of(1990, 1, 15);

    assertThatThrownBy(() -> Password.of(rawPassword, birthday))
        .isInstanceOf(CoreException.class);
}
```

**통합 테스트 (Integration Test)** — Service + Repository + DB를 함께 검증

```java
@SpringBootTest
class UserServiceIntegrationTest {
    @Autowired
    private UserService userService;

    @Test
    void 회원가입_후_조회하면_동일한_사용자가_반환된다() {
        userService.signUp(new SignUpCommand("testuser", "Test1234!@", ...));
        User found = userService.findByLoginId("testuser");
        assertThat(found.getLoginId()).isEqualTo("testuser");
    }
}
```

**E2E 테스트 (End-to-End Test)** — 실제 HTTP 요청으로 전체 흐름 검증

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserApiE2ETest {
    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void 회원가입_성공시_201_Created를_반환한다() {
        Map<String, Object> request = Map.of("loginId", "newuser", ...);
        ResponseEntity<String> response = restTemplate.postForEntity(
            "/api/v1/users/sign-up", request, String.class
        );
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    }
}
```

| 구분 | 단위 테스트 | 통합 테스트 | E2E 테스트 |
|------|-----------|-----------|-----------|
| 실행 속도 | ~ms | ~초 | ~분 |
| 피드백 속도 | 즉시 | 느림 | 매우 느림 |
| 실패 시 원인 파악 | 쉬움 | 보통 | 어려움 |
| 유지보수 비용 | 낮음 | 보통 | 높음 |

E2E 테스트가 실패하면 Controller → Service → Repository → DB 어디에서 문제가 생겼는지 추적해야 한다. 단위 테스트가 실패하면 "어떤 클래스의 어떤 메서드"인지 바로 안다.

**"넓게 한 번 검증"보다 "좁게 여러 번 검증"이 디버깅할 때 압도적으로 유리하다**는 걸 깨달았다.

> E2E 테스트만 잔뜩 있고 단위 테스트가 없는 역삼각형 구조는, 테스트 실행에 30분이 걸리고 실패해도 원인을 못 찾는다. 결국 테스트를 안 돌리게 된다.

---

## 3. Test Doubles — Mock만 알고 있었는데, 다섯 종류가 있었다

### 중점 학습 내용

단위 테스트에서 외부 의존성을 대체하는 객체를 통칭해서 **Test Doubles**라고 부른다.

다섯 가지 종류가 있다.

### Dummy — 전달만 되고, 실제로 사용되지 않는 객체

메서드 시그니처를 맞추기 위해 넣기만 하고 실제로 호출되지 않는 객체다.

```java
@Test
void 사용자_이름을_반환한다() {
    Logger dummyLogger = mock(Logger.class);
    UserService service = new UserService(repository, encoder, dummyLogger);

    User user = service.find(1L);
    assertThat(user.getName()).isEqualTo("홍길동");
}
```

역할은 단순하다. **자리를 채울 뿐, 아무 동작도 하지 않는다.**

### Stub — 미리 정해진 값을 반환하는 객체

호출되면 **미리 설정해둔 고정 값을 반환**한다. 실제 로직은 실행하지 않는다.

```java
@Test
void 존재하는_사용자를_조회하면_사용자_정보를_반환한다() {
    // Stub — findByLoginId를 호출하면 미리 만든 User를 반환한다
    UserRepository stubRepository = mock(UserRepository.class);
    User expectedUser = new User("testuser", "encoded", "홍길동", ...);
    when(stubRepository.findByLoginId("testuser")).thenReturn(expectedUser);

    UserService service = new UserService(stubRepository, encoder);

    // Act
    User result = service.findByLoginId("testuser");

    // Assert — 반환된 값을 검증한다 (상태 검증)
    assertThat(result.getName()).isEqualTo("홍길동");
}
```

핵심은 **"어떤 입력이 들어오면 어떤 출력을 줄 것인가"를 미리 정의**하는 것이다. 테스트가 외부 환경(DB, API)에 의존하지 않게 만들어준다.

### Spy — 실제 객체처럼 동작하면서 호출 기록을 남기는 객체

실제 메서드를 호출하되, **어떤 메서드가 몇 번 호출되었고 어떤 인자가 전달되었는지 기록**한다.

```java
@Test
void 회원가입_시_이벤트가_발행된다() {
    // Spy — 실제 EventPublisher를 감싸서 호출 기록을 추적한다
    EventPublisher realPublisher = new EventPublisher();
    EventPublisher spyPublisher = spy(realPublisher);

    UserService service = new UserService(repository, encoder, spyPublisher);
    service.signUp(new SignUpCommand("testuser", ...));

    // 실제로 이벤트가 발행되었는지 확인
    verify(spyPublisher).publish(any(UserSignedUpEvent.class));
}
```

Mock과 비슷해 보이지만 차이가 있다. **Spy는 실제 객체의 동작을 유지하면서 관찰만 추가**한다. Mock은 동작 자체를 대체한다.

### Mock — 기대하는 호출을 미리 정의하고, 호출 여부를 검증하는 객체

Mock은 **"이 메서드가 이 인자로 호출되어야 한다"는 기대를 설정하고, 실제로 그렇게 호출되었는지 검증**한다.

```java
@Test
void 비밀번호_변경_시_암호화_후_저장한다() {
    // Mock — 기대하는 호출을 정의
    PasswordEncoder mockEncoder = mock(PasswordEncoder.class);
    UserRepository mockRepository = mock(UserRepository.class);

    User user = new User("testuser", "old_encoded", ...);
    when(mockRepository.find(1L)).thenReturn(user);
    when(mockEncoder.encode("NewPass1!@")).thenReturn("new_encoded");
    when(mockEncoder.matches("OldPass1!@", "old_encoded")).thenReturn(true);

    UserService service = new UserService(mockRepository, mockEncoder);
    service.changePassword(1L, "OldPass1!@", "NewPass1!@");

    // 행위 검증 — encode가 새 비밀번호로 호출되었는가
    verify(mockEncoder).encode("NewPass1!@");
}
```

### Fake — 실제 구현의 단순화된 버전

Fake는 **실제로 동작하는 구현체**이지만, 프로덕션에서는 쓸 수 없는 단순한 버전이다. 대표적으로 In-Memory Repository가 있다.

```java
// 실제 DB 대신 메모리에 저장하는 Fake 구현체
public class FakeUserRepository implements UserRepository {
    private final Map<Long, User> store = new HashMap<>();
    private long sequence = 1L;

    @Override
    public User save(User user) {
        user.setId(sequence++);
        store.put(user.getId(), user);
        return user;
    }

    @Override
    public User find(Long id) {
        return store.get(id);
    }

    @Override
    public User findByLoginId(String loginId) {
        return store.values().stream()
            .filter(u -> u.getLoginId().equals(loginId))
            .findFirst()
            .orElse(null);
    }

    @Override
    public boolean existsByLoginId(String loginId) {
        return store.values().stream()
            .anyMatch(u -> u.getLoginId().equals(loginId));
    }
}
```

```java
@Test
void 회원가입_후_같은_아이디로_가입하면_예외가_발생한다() {
    // Fake — 실제처럼 동작하지만 DB가 필요 없다
    UserRepository fakeRepository = new FakeUserRepository();
    UserService service = new UserService(fakeRepository, encoder);

    service.signUp(new SignUpCommand("testuser", ...));

    assertThatThrownBy(() -> service.signUp(new SignUpCommand("testuser", ...)))
        .isInstanceOf(CoreException.class);
}
```

### Mock, Stub, Spy가 도대체 뭐가 다른 건지

Mockito에서 `mock()`으로 만든 객체에 `when().thenReturn()`을 쓰면 그게 Stub인 건지 Mock인 건지, `verify()`를 쓰면 Spy인 건지 Mock인 건지 구분이 안 됐다.

실제로 Mockito의 `mock()` 하나로 Stub, Mock, Spy 역할을 모두 할 수 있기 때문에 더 혼란스러웠다.

> **Stub을 기본으로 쓰고, Mock은 꼭 필요할 때만.** `when().thenReturn()`으로 원하는 상황을 만들어 놓고, 결과(상태)를 검증하는 것이 리팩토링에 강한 테스트를 만든다. `verify()`를 남발하면 구현이 바뀔 때마다 테스트가 깨진다.

Fake는 **테스트가 복잡해지거나 Stub으로 설정할 게 너무 많을 때** 유용했다. In-Memory Repository 하나 만들어두면 여러 테스트에서 재사용할 수 있고, 실제 DB 없이도 Service 계층을 온전히 테스트할 수 있다.

---

## 4. 단위 테스트의 종류 — Test Doubles를 활용한 검증 방식

### 중점 학습 내용

Test Doubles라는 도구를 알게 된 뒤, 단위 테스트를 검증 대상과 목적에 따라 분류할 수 있다는 걸 배웠다.

**상태 검증 테스트** — 메서드 실행 후 객체의 상태가 기대대로 변했는지 확인

```java
@Test
void 장바구니에_상품을_추가하면_수량이_증가한다() {
    Cart cart = new Cart();
    cart.addItem(new Item("상품A", 1000));

    assertThat(cart.getItemCount()).isEqualTo(1);
    assertThat(cart.getTotalPrice()).isEqualTo(1000);
}
```

**행위 검증 테스트** — Mock을 활용해, 특정 메서드가 호출되었는지 확인

```java
@Test
void 회원가입_시_비밀번호를_암호화하여_저장한다() {
    PasswordEncoder encoder = mock(PasswordEncoder.class);       // Mock
    UserRepository repository = mock(UserRepository.class);      // Mock
    when(encoder.encode("Test1234!@")).thenReturn("encoded_password"); // Stub처럼 사용

    UserService service = new UserService(repository, encoder);
    service.signUp(new SignUpCommand("user1", "Test1234!@", ...));

    verify(encoder).encode("Test1234!@");                        // Mock으로 행위 검증
    verify(repository).save(argThat(user ->
        user.getPassword().equals("encoded_password")
    ));
}
```

**예외 검증 테스트** — Stub으로 상황을 만들고, 비정상 입력에서 올바른 예외가 발생하는지 확인

```java
@Test
void 존재하지_않는_사용자의_비밀번호를_변경하면_예외가_발생한다() {
    UserRepository repository = mock(UserRepository.class);
    when(repository.find(999L)).thenReturn(null);                // Stub — null 반환 상황 설정

    UserService service = new UserService(repository, ...);

    assertThatThrownBy(() -> service.changePassword(999L, "old", "new"))
        .isInstanceOf(CoreException.class)
        .hasFieldOrPropertyWithValue("errorType", ErrorType.NOT_FOUND);
}
```

**경계값 테스트** — 입력값의 경계에서 올바르게 동작하는지 확인

```java
@Nested
@DisplayName("비밀번호 길이 경계값 테스트")
class PasswordLengthBoundary {
    private final LocalDate birthday = LocalDate.of(1990, 1, 15);

    @Test
    void 길이가_7자이면_실패한다() {
        assertThatThrownBy(() -> Password.of("Te1!abc", birthday))
            .isInstanceOf(CoreException.class);
    }

    @Test
    void 길이가_8자이면_성공한다() {
        Password password = Password.of("Te1!abcd", birthday);
        assertThat(password).isNotNull();
    }

    @Test
    void 길이가_16자이면_성공한다() {
        Password password = Password.of("Te1!abcdEf2@ghij", birthday);
        assertThat(password).isNotNull();
    }

    @Test
    void 길이가_17자이면_실패한다() {
        assertThatThrownBy(() -> Password.of("Te1!abcdEf2@ghijk", birthday))
            .isInstanceOf(CoreException.class);
    }
}
```

**파라미터화 테스트** — 같은 검증 로직을 여러 입력값에 반복 실행

```java
@ParameterizedTest
@ValueSource(strings = {
    "abcdefgh",       // 대문자 없음
    "ABCDEFGH",       // 소문자 없음
    "Abcdefgh",       // 숫자 없음
    "Abcdef1g",       // 특수문자 없음
})
@DisplayName("비밀번호 형식이 올바르지 않으면 예외가 발생한다")
void 비밀번호_형식_검증(String invalidPassword) {
    LocalDate birthday = LocalDate.of(1990, 1, 15);

    assertThatThrownBy(() -> Password.of(invalidPassword, birthday))
        .isInstanceOf(CoreException.class);
}
```

Test Doubles 분류와 연결지어 기준을 세웠다.

- **기본은 상태 검증 (Stub 활용)**이다. 결과(상태)만 맞으면 구현이 바뀌어도 테스트가 통과한다. 리팩토링에 강하다.
- **행위 검증 (Mock 활용)은 "반드시 호출되어야 하는 비즈니스 요구사항"이 있을 때만** 사용한다. "알림을 반드시 보내야 한다", "로그를 반드시 남겨야 한다" 같은 경우.

경계값 테스트는 **"통과하는 첫 번째 값과 마지막 값"과 "통과하는 값들의 앞, 뒤 값들"을 쌍으로** 작성해야 한다.

---

## 마무리 — 이번 주 학습 요약

| 학습 주제 | 헷갈렸던 점 | 이해하게 된 계기 |
|----------|-----------|---------------|
| TDD | 테스트 기법이라고 생각 | 테스트를 먼저 쓰면 인터페이스 설계가 먼저 이루어진다는 걸 체감 |
| 테스트 피라미드 | E2E 하나면 되지 않나? | 실패 시 원인 파악 난이도를 비교해보니 단위 테스트의 가치가 명확 |
| 단위 테스트 종류 | 상태 vs 행위 검증 구분 | "리팩토링했을 때 깨지는가"를 기준으로 나누니 명확 |
| Test Doubles | Mock, Stub, Spy 구분 | "상태를 검증하면 Stub, 행위를 검증하면 Mock"으로 정리 |

이번 주의 가장 큰 깨달음은 이거다.

> **테스트는 "코드가 동작하는지 확인하는 도구"가 아니라 "코드가 어떤 의도로 만들어졌는지 표현하는 문서"다.** 좋은 테스트를 작성하면 누구든 이 코드가 무엇을 해야 하는지 테스트만 보고 이해할 수 있다.
