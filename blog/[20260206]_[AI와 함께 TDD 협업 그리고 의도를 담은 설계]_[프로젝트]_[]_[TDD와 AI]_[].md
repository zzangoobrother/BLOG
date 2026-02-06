# AI와 함께 TDD 협업 그리고 의도를 담은 설계를 구현하며 느낀 것들

## 들어가며

TDD를 하면서 느낀 것들과 AI를 사용하면서 단순한 사용이 아닌 어떻게 함께 했는지를 정리해볼려고 한다.  
회원가입, 내 정보 조회, 비밀번호 변경. 기능만 보면 단순하다. 하지만 이번 프로젝트에서 중요했던 건 **"무엇을 만들었는가"가 아니라 "어떻게 만들었는가"**였다.

1. **단순한 기능에도 의도를 담는 설계** — Value Object, 레이어 분리, 검증 로직의 위치
2. **AI를 "도구"가 아닌 "페어 프로그래밍 파트너"로 활용한 경험** — CLAUDE.md, plan.md 기반 증강 코딩

## 프로젝트 구조: 왜 이렇게 나눴는가

### 멀티모듈이 필요했던 이유

프로젝트는 다음과 같은 구조를 갖는다.

```
loop-pack-be-l2-vol3-kotlin/
├── apps/
│   ├── commerce-api         # REST API
│   ├── commerce-batch       # 배치 처리
│   └── commerce-streamer    # Kafka Consumer
├── modules/                 # 재사용 가능한 인프라 설정
│   ├── jpa/
│   ├── redis/
│   └── kafka/
└── supports/                # 부가 기능
    ├── jackson/
    ├── logging/
    └── monitoring/
```

API 서버, 배치, 스트리머가 공존하는 구조다. 세 애플리케이션이 같은 JPA 설정, 같은 Entity를 사용해야 한다면? 코드를 복사하는 건 답이 아니다. `modules/jpa`에 BaseEntity와 JPA 설정을 공유하고, 각 앱은 자신의 비즈니스 로직에만 집중하도록 했다.

> 멀티모듈은 "코드 재사용"보다 "관심사 분리"에 가깝다.
> 같은 코드를 쓰는 게 목적이 아니라, 각 앱이 자기 책임에만 집중할 수 있게 하는 것이다.

### 레이어드 아키텍처 — 각 계층의 역할

```
interfaces/api/    → Controller, ApiSpec(OpenAPI), Dto
application/       → Facade(오케스트레이션), Info(레이어간 데이터 전달)
domain/            → Entity, Service(@Component), Repository(인터페이스)
infrastructure/    → RepositoryImpl(구현체), JpaRepository
support/error/     → CoreException, ErrorType
```

요청 흐름은 이렇다.

```
Controller → Facade → Service → Repository(interface) → RepositoryImpl → JpaRepository
```

여기서 중요한 건 **Repository가 인터페이스**라는 것이다. 도메인 계층에 `UserRepository` 인터페이스를 두고, `infrastructure`에서 `UserRepositoryImpl`이 이를 구현한다. JPA에 직접 의존하지 않는 도메인 계층. 테스트할 때 Mock으로 갈아끼우기도 쉽다.

```kotlin
// domain/user/UserRepository.kt — 도메인이 정의하는 "나는 이런 기능이 필요해"
interface UserRepository {
    fun find(id: Long): User?
    fun findByLoginId(loginId: String): User?
    fun existsByLoginId(loginId: String): Boolean
    fun save(user: User): User
}
```

```kotlin
// infrastructure/user/UserRepositoryImpl.kt — 인프라가 구현하는 "이렇게 해줄게"
@Component
class UserRepositoryImpl(
    private val userJpaRepository: UserJpaRepository,
) : UserRepository {
    override fun find(id: Long): User? {
        return userJpaRepository.findByIdOrNull(id)
    }
    // ...
}
```

Facade 계층이 있는 이유도 명확하다. 서비스 간 조합이 필요한 시점이 오면 Facade가 오케스트레이션을 담당한다. 지금은 UserService 하나만 호출하지만, 나중에 포인트 적립, 알림 발송 같은 로직이 추가될 때 Service끼리 직접 호출하지 않고 Facade에서 조율할 수 있다.

## 의도가 드러나는 설계: Value Object

이번 프로젝트에서 가장 신경 쓴 부분이다.

### 문제: 검증 로직이 어디에 있어야 하는가

회원가입 시 비밀번호 검증 규칙은 다음과 같다.

- 8~16자
- 영문 대소문자 + 숫자 + 특수문자 모두 포함
- 생년월일(yyyyMMdd) 미포함

이 검증을 어디서 할 것인가? Controller? Service? Entity?

**선택지 1: Service에 두기**

처음에는 Service에 두는 게 자연스러워 보였다. 회원가입 흐름을 처리하는 `signUp()` 메서드 안에서 비밀번호 형식을 검증하고, 생년월일 포함 여부를 확인하는 식이다. 하지만 이렇게 하면 검증 로직이 서비스에 흩어진다. `signUp()`에도 비밀번호 검증이 있고, `changePassword()`에도 같은 검증이 있어야 한다. "비밀번호란 무엇인가"라는 도메인 지식이 서비스 메서드 곳곳에 묻혀버린다.

**선택지 2: Entity의 init 블록에 두기**

그렇다면 Entity에 직접 검증을 넣으면 어떨까? User Entity의 `init` 블록에서 비밀번호 정규식을 검증하는 방법이다. 객체 생성 시점에 검증이 강제되니 유효하지 않은 User가 존재할 수 없다는 장점이 있다. 하지만 문제가 있다. 비밀번호는 **암호화된 상태로 Entity에 저장**된다. Entity가 받는 password는 이미 BCrypt나 PBKDF2로 인코딩된 문자열이다. 여기서 "8~16자, 대소문자 포함" 같은 원문 비밀번호 규칙을 검증하는 건 맞지 않다. 또한 생년월일 포함 여부 검증을 하려면 Entity가 원문 비밀번호를 알아야 하는데, 암호화 후에는 원문을 알 수 없다.

> Entity에서 검증할 수 있는 것과 없는 것을 구분해야 한다. loginId의 형식(영문+숫자), name의 빈 값 여부, email 형식 — 이런 건 Entity가 저장하는 값 그 자체에 대한 검증이라 init 블록에서 할 수 있다. 하지만 비밀번호처럼 "원문 → 암호화 → 저장"이라는 변환 과정이 있는 필드는, 변환 전 시점에서 검증해야 한다.

**선택지 3: Value Object로 분리하기 (최종 선택)**

결국 검증을 담당하는 별도의 객체가 필요했다. "원문 비밀번호"라는 개념 자체를 객체로 만들고, 이 객체가 생성되는 시점에 모든 규칙을 검증하는 것이다.

### 해결: Value Object로 도메인 지식 캡슐화

**Password**, **LoginId**, **Email**, **MaskedName** — 4개의 Value Object를 만들었다.

```kotlin
class Password private constructor(val value: String) {
    companion object {
        private val PASSWORD_PATTERN =
            Regex("^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@\$!%*?&#])[A-Za-z\\d@\$!%*?&#]{8,16}$")

        fun of(rawPassword: String, birthday: LocalDate): Password {
            validateFormat(rawPassword)
            validateNotContainsBirthday(rawPassword, birthday)
            return Password(rawPassword)
        }

        private fun validateFormat(password: String) {
            if (!PASSWORD_PATTERN.matches(password)) {
                throw CoreException(
                    ErrorType.BAD_REQUEST,
                    "비밀번호는 8~16자의 영문 대소문자, 숫자, 특수문자를 포함해야 합니다.",
                )
            }
        }

        private fun validateNotContainsBirthday(password: String, birthday: LocalDate) {
            val birthdayString = birthday.toString().replace("-", "")
            if (password.contains(birthdayString)) {
                throw CoreException(
                    ErrorType.BAD_REQUEST,
                    "비밀번호에 생년월일을 포함할 수 없습니다.",
                )
            }
        }
    }
}
```

**`private constructor`로 생성자를 잠그고 `of` 팩토리 메서드만 열어둔 것**이 핵심이다. Password 객체가 존재한다는 것 자체가 "검증을 통과했다"는 증거가 된다. 별도로 `isValid()` 같은 메서드를 호출할 필요가 없다.

LoginId도 마찬가지다.

```kotlin
class LoginId private constructor(val value: String) {
    companion object {
        private val LOGIN_ID_PATTERN = Regex("^[a-zA-Z0-9]+$")

        fun of(value: String): LoginId {
            validate(value)
            return LoginId(value)
        }

        private fun validate(value: String) {
            if (value.isBlank()) {
                throw CoreException(ErrorType.BAD_REQUEST, "로그인 ID는 비어있을 수 없습니다.")
            }
            if (!LOGIN_ID_PATTERN.matches(value)) {
                throw CoreException(ErrorType.BAD_REQUEST, "로그인 ID는 영문과 숫자만 사용할 수 있습니다.")
            }
        }
    }
}
```

이렇게 하면 User Entity의 `init` 블록이 깔끔해진다.

```kotlin
@Entity
@Table(name = "users")
class User(
    loginId: String,
    password: String,
    name: String,
    birthday: LocalDate,
    email: String,
) : BaseEntity() {
    // ...
    init {
        LoginId.of(loginId)
        validateName(name)
        Email.of(email)
    }
}
```

Entity가 생성되는 순간 모든 검증이 완료된다. "User 객체가 존재한다 = 모든 필드가 유효하다"는 불변식이 성립한다.

### MaskedName — 응답 시점의 변환도 도메인 지식이다

내 정보 조회 API에서 이름을 마스킹해야 한다. "홍길동" → "홍길*". 이것도 Value Object로 만들었다.

```kotlin
class MaskedName private constructor(val value: String) {
    companion object {
        fun from(name: String): MaskedName {
            if (name.isEmpty()) {
                throw CoreException(ErrorType.BAD_REQUEST, "이름은 비어있을 수 없습니다.")
            }
            val masked = name.dropLast(1) + "*"
            return MaskedName(masked)
        }
    }
}
```

마스킹 규칙이 바뀌면? MaskedName 하나만 수정하면 된다. Controller에서 문자열을 직접 조작하는 것보다 의도가 명확하다.

### PasswordEncoder — 도메인이 구현을 모르게 하기

비밀번호 암호화도 같은 원리다. `PasswordEncoder` 인터페이스를 **도메인 계층**에 정의했다.

```kotlin
// domain/user/PasswordEncoder.kt — 도메인이 정의하는 "나는 이런 기능이 필요해"
interface PasswordEncoder {
    fun encode(rawPassword: String): String
    fun matches(rawPassword: String, encodedPassword: String): Boolean
}
```

도메인은 "비밀번호를 암호화하고 검증하는 기능이 필요하다"만 안다. PBKDF2든 BCrypt든 구현체를 모른다. 실제 구현은 infrastructure에 있고, `@Component`로 주입될 뿐이다.

```kotlin
// infrastructure/user/Pbkdf2PasswordEncoder.kt — 인프라가 구현하는 "이렇게 해줄게"
@Component
class Pbkdf2PasswordEncoder : PasswordEncoder {
    override fun encode(rawPassword: String): String { /* ... */ }
    override fun matches(rawPassword: String, encodedPassword: String): Boolean { /* ... */ }
}
```

앞서 본 Repository 인터페이스와 같은 구조다. 도메인이 "무엇이 필요한가"를 정의하고, 인프라가 "어떻게 제공할 것인가"를 구현한다. 나중에 BCrypt로 바꿔야 한다면? `BcryptPasswordEncoder`를 만들어서 `@Component`를 붙이면 된다. 도메인 코드는 한 줄도 안 바뀐다.

> 이 경계가 있어야 구현을 교체할 때 도메인이 흔들리지 않는다. "어떤 알고리즘을 쓰는가"는 인프라의 관심사지, 도메인의 관심사가 아니다.

## 인증 구현: Filter + ArgumentResolver 조합

인증은 커스텀 헤더(`X-Loopers-LoginId`, `X-Loopers-LoginPw`) 기반이다. Spring Security를 쓰지 않고 직접 구현했다.

```kotlin
@Component
@Order(1)
class AuthenticationFilter(
    private val userService: UserService,
    private val objectMapper: ObjectMapper,
) : Filter {

    override fun doFilter(request: ServletRequest, response: ServletResponse, chain: FilterChain) {
        val httpRequest = request as HttpServletRequest

        try {
            if (requiresAuthentication(httpRequest)) {
                val loginId = httpRequest.getHeader(LOGIN_ID_HEADER)
                val password = httpRequest.getHeader(LOGIN_PW_HEADER)

                if (loginId.isNullOrBlank() || password.isNullOrBlank()) {
                    writeErrorResponse(httpResponse, ErrorType.UNAUTHORIZED, "인증 헤더가 필요합니다.")
                    return
                }

                val user = userService.authenticate(loginId, password)
                httpRequest.setAttribute(AUTHENTICATED_USER_ATTRIBUTE, user)
            }

            chain.doFilter(request, response)
        } catch (e: CoreException) {
            writeErrorResponse(httpResponse, e.errorType, e.message ?: e.errorType.message)
        }
    }
}
```

Filter에서 인증된 User 객체를 `request.setAttribute`에 저장하고, `AuthenticatedUserArgumentResolver`가 이를 꺼내서 Controller 파라미터에 주입한다.

```kotlin
// Controller에서는 이렇게 사용한다
@GetMapping("/me")
override fun getMe(
    @AuthenticatedUser user: User,  // ArgumentResolver가 자동 주입
): ApiResponse<UserDto.MeResponse> {
    return userFacade.getMe(user)
        .let { UserDto.MeResponse.from(it) }
        .let { ApiResponse.success(it) }
}
```

Controller가 인증 로직을 전혀 모른다. `@AuthenticatedUser`만 붙이면 된다. 관심사가 깔끔하게 분리되어 있다.

## 비밀번호 변경: 작은 기능에 담긴 설계 의도

비밀번호 변경 API 하나에도 여러 설계 결정이 녹아 있다.

```kotlin
@Transactional
fun changePassword(userId: Long, currentPassword: String, newPassword: String) {
    val user = userRepository.find(userId)
        ?: throw CoreException(ErrorType.NOT_FOUND, "사용자를 찾을 수 없습니다.")

    // 1. 현재 비밀번호 검증
    verifyPassword(currentPassword, user.password)

    // 2. 현재 비밀번호와 새 비밀번호 동일 여부 확인
    if (currentPassword == newPassword) {
        throw CoreException(ErrorType.BAD_REQUEST, "새 비밀번호는 현재 비밀번호와 달라야 합니다.")
    }

    // 3. 새 비밀번호 형식 검증 + 암호화
    val encodedPassword = encodePassword(newPassword, user.birthday)

    // 4. Entity에 변경 위임
    user.changePassword(encodedPassword)
}
```

여기서 주목할 점.

> **비밀번호 검증과 인코딩 로직이 Service에 있다.** 처음에는 User Entity 안에 `changePassword(rawPassword, encoder)` 형태로 구현했다. 하지만 리팩토링 과정에서 Entity가 `PasswordEncoder`를 알아야 할 이유가 없다는 결론에 도달했다. Entity는 "이미 암호화된 비밀번호를 받아서 저장"하는 역할만 한다.

```kotlin
// User Entity — 최소한의 책임만 가진다
fun changePassword(encodedNewPassword: String) {
    this.password = encodedNewPassword
}
```

이 리팩토링은 커밋 히스토리에도 드러난다.

<a href='https://github.com/zzangoobrother/loop-pack-be-l2-vol3-kotlin/pull/10/changes/a7fec47ce603e0df3bcfcc393a288203aba76e29' target='_blank'>refactor: 비밀번호 검증/인코딩 로직을 User에서 UserService로 이동</a>
<br>
<a href='https://github.com/zzangoobrother/loop-pack-be-l2-vol3-kotlin/pull/10/changes/0d8b373b3e2a1238c42c473e3ff189577425729a' target='_blank'>feat: 비밀번호 변경 API 구현</a>

![image_20260206_1.png](img/image_20260206_1.png)

그리고 구조적 변경과 행위적 변경을 분리해서 커밋했다. Tidy First 원칙이다.

## AI 증강 코딩: CLAUDE.md와 plan.md 활용법

이번 프로젝트에서 가장 실험적이었던 부분이다. Claude Code를 단순한 코드 생성기가 아닌, **TDD 사이클을 함께 돌리는 페어 프로그래밍 파트너**로 활용했다.

### CLAUDE.md — AI에게 "프로젝트 맥락"을 주입하기

CLAUDE.md는 AI가 프로젝트에 참여할 때 읽는 설명서다. 여기에 아키텍처, 코딩 컨벤션, 개발 방법론을 모두 담았다.

핵심은 **"AI가 임의로 판단하지 않게 하는 것"**이었다.

```markdown
## 증강 코딩 원칙

- **대원칙**: 방향성 및 주요 의사 결정은 개발자에게 제안만 하며, 최종 승인된 사항을 기반으로 작업 수행
- **임의 작업 금지**: 반복적 동작, 요청하지 않은 기능 구현, 테스트 삭제를 임의로 진행하지 않는다
- **설계 주도권**: AI는 임의판단하지 않고 방향성을 제안할 수 있으나, 개발자 승인 후 수행
```

이 원칙 없이 AI를 쓰면 어떻게 될까? AI는 "도움이 된다고 생각하는" 코드를 마구 생성한다. 요청하지 않은 에러 핸들링을 추가하고, 불필요한 테스트를 만들고, 프로젝트 컨벤션과 다른 방식으로 코드를 작성한다.

> AI를 잘 쓰려면 "하지 말 것"을 명확히 해야 한다. "잘 해줘"보다 "이것만 해줘"가 훨씬 효과적이다.

### plan.md — 단위 작업을 명시하기

plan.md에는 현재 구현해야 할 기능의 단위 작업 목록을 작성했다.

```markdown
# 기능명: 5.3 비밀번호 수정 API

## 구현 계획

### 1. 도메인 모델 (User Entity)
- [ ] 현재 비밀번호와 새 비밀번호가 동일하면 예외 발생
- [ ] User.changePassword 메서드 구현

### 2. 서비스 (UserService)
- [ ] 현재 비밀번호 검증 실패 시 UNAUTHORIZED 예외 발생
- [ ] UserService.changePassword 메서드 구현

### 3. Facade (UserFacade)
- [ ] UserFacade.changePassword 메서드 구현

### 4. API (Controller, DTO, ApiSpec)
- [ ] UserDto.ChangePasswordRequest 추가
- [ ] UserController.changePassword 엔드포인트 구현

### 5. E2E 테스트
- [ ] 비밀번호 변경 성공 → 200 OK
- [ ] 현재 비밀번호 불일치 → 401 UNAUTHORIZED
- [ ] 새 비밀번호 형식 오류 → 400 BAD_REQUEST
```

AI에게 `red`, `green`, `refactor` 커맨드를 사용해 TDD 사이클의 각 단계를 명시적으로 지시한다. `red`는 실패하는 테스트 작성, `green`은 테스트를 통과시키는 최소 구현, `refactor`는 코드 개선이다.

### 실제 워크플로우

내가 AI와 작업한 시나리오는 이렇다.

**1단계: PRD 작성 (AI와 함께, 내가 검토한다)**

기능 명세를 PRD.md에 작성한다. 어떤 API가 필요한지, 검증 규칙은 뭔지, 에러 코드는 뭔지 정의하는 과정을 AI와 함께 진행한다. 내가 필요한 요구사항을 전달하면 AI가 초안을 작성하고, 나는 빠진 부분이 없는지, 검증 규칙이 맞는지, 에러 케이스가 충분한지를 검토한다. **완벽할 때까지 수정을 반복한다.** 요구사항의 방향은 개발자가 잡지만, 구체화하는 과정에서 AI의 도움을 받으면 놓치기 쉬운 엣지 케이스나 명세의 빈틈을 빠르게 채울 수 있다.

**2단계: 구현 계획 작성 (AI에게 제안받고, 내가 확정한다)**

PRD를 기반으로 AI에게 구현 계획을 요청한다. AI는 plan.md 초안을 제안하고, 나는 순서를 조정하거나 항목을 추가/삭제한다. 예를 들어, AI가 "Service부터 구현하겠습니다"라고 제안하면 "도메인 모델부터 시작해"라고 방향을 잡아준다.

**3단계: TDD 사이클 (AI와 함께)**

```
나: "red"
AI: plan.md의 첫 번째 미완료 항목 확인
    → 실패하는 테스트 작성
    → 테스트 실행, 실패 확인
나: 테스트 코드 리뷰
나: "green"
AI: 테스트를 통과시키는 최소 구현
    → 테스트 실행, 통과 확인
나: 구현 코드 리뷰
나: "refactor"
AI: 중복 제거, 이름 개선 등
    → 전체 테스트 실행, 통과 확인
```

이 사이클을 반복한다. 핵심은 **커맨드 하나에 한 단계만 처리한다**는 것이다. `red`를 보내면 테스트만 작성하고 멈추고, `green`을 보내면 구현만 하고 멈춘다. AI가 한 번에 여러 단계를 건너뛰지 않으니 각 단계마다 리뷰할 수 있고, 문제가 생겼을 때 원인을 찾기 쉽다.

**4단계: 커밋 (AI가 하되, 규칙을 따른다)**

```markdown
## 커밋 규칙
- 모든 테스트가 통과하고, 린터 경고가 없을 때만 커밋
- 하나의 논리적 작업 단위로 커밋
- 구조적 변경과 행위적 변경을 같은 커밋에 섞지 않는다
```

실제 커밋 히스토리를 보면 이 원칙이 지켜지고 있다.

<a href='https://github.com/zzangoobrother/loop-pack-be-l2-vol3-kotlin/pull/10/changes/74ac4b89bc9baf29399d948cdb40cf5652623671' target='_blank'>refactor: User, UserService 검증 로직을 Value Object로 위임</a>
<br>
<a href='https://github.com/zzangoobrother/loop-pack-be-l2-vol3-kotlin/pull/10/changes/807b504f7bd76bf3586081b1f64d396f5b389aea' target='_blank'>feat: Password, LoginId, Email Value Object 구현</a>
<br>
<a href='https://github.com/zzangoobrother/loop-pack-be-l2-vol3-kotlin/pull/10/changes/9b91265162935eb9aded4e9e248c383db3c89bb7' target='_blank'>feat: 회원가입 API 구현</a>
<br>
<a href='https://github.com/zzangoobrother/loop-pack-be-l2-vol3-kotlin/pull/10/changes/a9405e1d753974c22f505a48b5f83b8df23ce00d' target='_blank'>feat: 회원가입 시 비밀번호 암호화 기능 구현</a>

Value Object 도입(구조적 변경)과 기능 구현(행위적 변경)이 분리되어 있다.

### AI 활용에서 얻은 교훈

**잘 된 것:**
- CLAUDE.md에 아키텍처와 패턴을 명시하니 AI가 프로젝트 컨벤션을 정확히 따랐다
- plan.md로 작업 범위를 제한하니 AI가 "요청하지 않은 기능"을 만들지 않았다
- TDD 사이클에서 테스트 작성 → 최소 구현 흐름이 일관되게 유지되었다
- 리팩토링 시 "비밀번호 검증 로직을 Entity에서 Service로 이동"같은 구조적 변경을 AI가 제안했고, 설득력이 있어서 수용했다

**주의할 것:**
- AI가 작성한 코드를 **반드시 리뷰**해야 한다. 특히 테스트 코드에서 "실제로 검증하고 있는가"를 확인해야 한다. 테스트가 항상 통과하는 무의미한 테스트를 작성하는 경우가 있다
- AI는 "왜 이렇게 했는가"를 모른다. 코드를 생성할 수는 있지만 설계 의도는 개발자가 가져야 한다
- 커맨드만 반복하면 AI에게 주도권을 넘기는 것이 된다. 중간중간 방향을 잡아주는 게 중요하다

## 테스트 전략: 세 가지 레벨

### 단위 테스트 — Value Object와 Entity

```kotlin
@Nested
@DisplayName("Password 생성")
inner class CreatePassword {

    @Test
    fun `유효한 비밀번호로 생성할 수 있다`() {
        // Arrange
        val rawPassword = "Test1234!@"
        val birthday = LocalDate.of(1990, 1, 15)

        // Act
        val password = Password.of(rawPassword, birthday)

        // Assert
        assertThat(password.value).isEqualTo(rawPassword)
    }

    @Test
    fun `생년월일이 포함된 비밀번호는 생성할 수 없다`() {
        // Arrange
        val rawPassword = "Test19900115!@"
        val birthday = LocalDate.of(1990, 1, 15)

        // Act & Assert
        assertThatThrownBy { Password.of(rawPassword, birthday) }
            .isInstanceOf(CoreException::class.java)
    }
}
```

`@Nested` + `@DisplayName`으로 BDD 스타일을 구성하고, 3A(Arrange-Act-Assert) 원칙을 따랐다. Value Object가 검증을 책임지니 테스트도 Value Object 단위로 작성하면 된다.

### 통합 테스트 — Service 계층

Service 테스트에서는 실제 DB를 사용한다. TestContainers로 MySQL을 띄우고, `@AfterEach`에서 테이블을 비운다.

```kotlin
@AfterEach
fun tearDown() {
    databaseCleanUp.truncateAllTables()
}
```

### E2E 테스트 — API 레벨

```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserApiE2ETest {
    // TestRestTemplate으로 실제 HTTP 요청
}
```

E2E 테스트는 실제 서버를 띄우고 HTTP 요청을 보낸다. Controller → Facade → Service → Repository → DB 전체 흐름을 검증한다.

> 단위 테스트로 "각 부품이 잘 동작하는가"를 확인하고, E2E 테스트로 "부품이 조립됐을 때 잘 동작하는가"를 확인한다. 두 레벨 모두 필요하다.

## 마무리: 기능이 아니라 의도를 구현하다

이번 프로젝트에서 배운 것을 정리한다.

### 설계에 대해

- **Value Object는 도메인 지식을 캡슐화하는 최소 단위다.** 검증 로직을 Service에 흩뿌리지 않고, "비밀번호란 이런 것"이라는 정의를 한 곳에 모았다.
- **계층 간 역할을 명확히 하면, 변경의 영향 범위가 줄어든다.** Entity는 상태를 관리하고, Service는 비즈니스 흐름을 관리하고, Controller는 HTTP 요청/응답을 관리한다. 각자 자기 일만 한다.
- **구조적 변경과 행위적 변경을 분리하면, 코드 리뷰가 쉬워진다.** 커밋 하나에 리팩토링과 기능 추가가 섞여 있으면 리뷰어가 "이게 동작이 바뀐 건지, 코드만 바뀐 건지" 판단하기 어렵다.

### AI 활용에 대해

- **AI는 "방향"을 잡아주면 강력한 실행력을 보여준다.** CLAUDE.md로 맥락을 주고, plan.md로 범위를 제한하면 일관된 품질의 코드를 생성한다.
- **AI의 진짜 가치는 코드 생성이 아니라 "사이클 속도"에 있다.** Red-Green-Refactor 한 사이클을 수 분 안에 돌릴 수 있다. 이 속도가 모여서 하루에 수십 번의 TDD 사이클을 돌리게 된다.
- **하지만 설계 주도권은 반드시 개발자에게 있어야 한다.** AI는 "어떻게"는 잘하지만 "왜"는 모른다. "왜 Value Object로 분리하는가", "왜 이 검증이 Service가 아닌 도메인에 있어야 하는가"는 개발자가 결정해야 한다.

단순한 회원 API를 만들면서도, 의도를 담으면 생각할 점과 배울점이 많았다.

