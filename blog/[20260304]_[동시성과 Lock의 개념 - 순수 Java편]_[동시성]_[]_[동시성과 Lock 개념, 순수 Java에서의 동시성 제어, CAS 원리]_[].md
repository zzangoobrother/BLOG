# 동시성과 Lock의 개념 - 순수 Java편

## 들어가며

백엔드 개발자라면 피할 수 없는 주제가 있다. **동시성(Concurrency)**이다. 단일 스레드 환경에서는 문제없이 동작하던 코드가, 멀티스레드 환경에서는 예측 불가능한 결과를 만들어낸다. "분명 100번 차감했는데 왜 97개만 줄었지?" 같은 상황을 한 번이라도 겪어봤다면, 동시성 문제의 무서움을 알 것이다.

이 글은 시리즈의 1편으로, **순수 Java 관점**에서 동시성을 다룬다. DB Lock이나 Redis 분산락 같은 인프라 수준의 제어는 2편에서 별도로 다룰 예정이다.

이번 글에서 다루는 내용은 다음과 같다.

1. 동시성 문제는 왜 발생하는가
2. Java가 제공하는 동시성 제어 도구들
3. CAS(Compare-And-Swap) 원리와 Lock-Free 프로그래밍

---

## 동시성 문제는 왜 발생하는가

### 공유 자원과 경쟁 조건(Race Condition)

동시성 문제의 근본 원인은 단순하다. **여러 스레드가 하나의 공유 자원에 동시에 접근**하기 때문이다.

```java
public class Counter {
    private int count = 0;

    public void increment() {
        count++; // 이 한 줄이 문제다
    }

    public int getCount() {
        return count;
    }
}
```

`count++`는 코드상 한 줄이지만, 실제로는 세 단계로 실행된다.

```
1. count 값을 읽는다 (READ)
2. 읽은 값에 1을 더한다 (MODIFY)
3. 결과를 count에 쓴다 (WRITE)
```

두 스레드가 동시에 이 과정을 실행하면 어떻게 될까?

```
시간  스레드A              스레드B              count 실제값
──────────────────────────────────────────────────────────
t1   READ count = 0                            0
t2                        READ count = 0       0
t3   MODIFY 0 + 1 = 1                          0
t4                        MODIFY 0 + 1 = 1     0
t5   WRITE count = 1                           1
t6                        WRITE count = 1      1
```

두 스레드 모두 `increment()`를 호출했지만, count는 2가 아니라 **1**이다. 이것이 **경쟁 조건(Race Condition)**이다. 스레드의 실행 순서에 따라 결과가 달라지는 비결정적 상황을 의미한다.

### 가시성(Visibility) 문제

경쟁 조건만이 문제가 아니다. **가시성 문제**도 있다.

CPU는 성능을 위해 메인 메모리와 별도로 **CPU 캐시**를 사용한다. 각 스레드는 변수를 메인 메모리가 아닌 자신의 CPU 캐시에서 읽고 쓸 수 있다.

```
┌─────────────┐    ┌─────────────┐
│   스레드 A    │    │   스레드 B    │
│ ┌─────────┐ │    │ ┌─────────┐ │
│ │CPU 캐시  │ │    │ │CPU 캐시  │ │
│ │count = 0│ │    │ │count = 0│ │
│ └─────────┘ │    │ └─────────┘ │
└──────┬──────┘    └──────┬──────┘
       │                  │
       ▼                  ▼
┌──────────────────────────────┐
│        메인 메모리             │
│        count = 0             │
└──────────────────────────────┘
```

스레드 A가 `count`를 1로 바꿔도, 그 변경이 메인 메모리에 즉시 반영되지 않을 수 있다. 스레드 B는 여전히 자신의 캐시에 있는 `count = 0`을 읽는다. 이것이 **가시성 문제**다.

### 명령어 재배치(Instruction Reordering)

JVM과 CPU는 성능 최적화를 위해 명령어의 실행 순서를 재배치할 수 있다. 단일 스레드에서는 논리적 순서가 보장되지만, 멀티스레드에서는 문제가 된다.

```java
// 원래 코드
int a = 1;       // (1)
boolean flag = true;  // (2)

// JVM이 재배치할 수 있다
boolean flag = true;  // (2) 먼저 실행
int a = 1;       // (1) 나중에 실행
```

다른 스레드가 `flag == true`를 보고 `a`를 읽었는데, 아직 `a = 1`이 실행되지 않은 상태라면? **불완전한 데이터를 읽게 된다.** 이것을 방지하려면 **메모리 배리어(Memory Barrier)**가 필요하다.

---

## Java가 제공하는 동시성 제어 도구들

### 1. synchronized - 가장 기본적인 동기화

`synchronized`는 Java가 언어 수준에서 제공하는 동기화 메커니즘이다. **모니터 락(Monitor Lock)**을 기반으로 동작한다.

```java
public class SynchronizedCounter {
    private int count = 0;

    // 메서드 수준 동기화
    public synchronized void increment() {
        count++;
    }

    // 블록 수준 동기화
    public void decrement() {
        synchronized (this) {
            count--;
        }
    }

    public synchronized int getCount() {
        return count;
    }
}
```

#### 동작 원리

Java의 모든 객체는 내부에 **모니터(Monitor)**를 하나씩 가지고 있다. `synchronized`는 이 모니터를 획득하고 해제하는 방식으로 동기화를 보장한다.

```
스레드A: 모니터 획득 → 임계 영역 실행 → 모니터 해제
스레드B: 모니터 획득 대기... → 모니터 획득 → 임계 영역 실행 → 모니터 해제
```

`synchronized`는 세 가지를 동시에 보장한다.

1. **상호 배제(Mutual Exclusion)**: 한 번에 하나의 스레드만 임계 영역에 진입
2. **가시성 보장**: 락을 해제할 때 변경사항이 메인 메모리에 반영(flush), 락을 획득할 때 메인 메모리에서 최신값을 읽음(refresh)
3. **재배치 방지**: synchronized 블록 안팎으로 명령어가 이동하지 않음

#### 한계

```java
// 문제 1: 락을 기다리는 동안 인터럽트 불가
public synchronized void longRunningMethod() {
    // 매우 긴 작업...
    // 이 작업이 끝날 때까지 다른 스레드는 무한 대기
}

// 문제 2: 읽기-읽기도 블로킹
public synchronized int getCount() {
    return count; // 읽기만 하는데도 다른 읽기 스레드를 차단
}

// 문제 3: 공정성 보장 안 됨
// 어떤 스레드가 먼저 락을 획득할지 보장되지 않음
// 특정 스레드가 계속 락을 획득하지 못하는 기아(starvation) 발생 가능
```

`synchronized`는 단순하고 안전하지만, 세밀한 제어가 필요한 상황에서는 부족하다.

---

### 2. volatile - 가시성 보장

`volatile`은 **가시성 문제**를 해결하는 키워드다. 상호 배제는 제공하지 않는다.

```java
public class VolatileExample {
    private volatile boolean running = true;

    public void stop() {
        running = false; // 메인 메모리에 즉시 반영
    }

    public void run() {
        while (running) { // 항상 메인 메모리에서 읽음
            // 작업 수행
        }
        System.out.println("스레드 종료");
    }
}
```

`volatile`이 없으면 `run()` 메서드의 스레드가 `running`의 변경을 영원히 감지하지 못할 수 있다. CPU 캐시에 저장된 `running = true`만 계속 읽기 때문이다.

#### volatile이 보장하는 것

1. **가시성**: 변수의 읽기/쓰기가 항상 메인 메모리를 통해 이루어진다
2. **happens-before 관계**: volatile 쓰기 이전의 모든 변경사항은, volatile 읽기 이후에 보인다

```java
// volatile의 happens-before
int a = 0;
volatile boolean flag = false;

// 스레드 A
a = 42;           // (1)
flag = true;      // (2) volatile 쓰기 → (1)의 변경도 메인 메모리에 반영

// 스레드 B
if (flag) {       // (3) volatile 읽기 → a의 최신값도 보장
    System.out.println(a); // 반드시 42 출력
}
```

#### volatile만으로 부족한 경우

```java
private volatile int count = 0;

public void increment() {
    count++; // 여전히 안전하지 않다!
}
```

`volatile`은 **읽기와 쓰기 각각**의 원자성만 보장한다. `count++`처럼 "읽고-수정하고-쓰는" 복합 연산은 원자적이지 않기 때문에 여전히 경쟁 조건이 발생한다. 이런 경우에는 `synchronized`나 `Atomic` 클래스를 사용해야 한다.

#### 언제 volatile을 쓸까?

- **하나의 스레드만 쓰고, 여러 스레드가 읽는** 상황 (플래그 변수)
- 상태 전이가 **단순 대입**인 경우 (`flag = true/false`)
- 복합 연산이 필요하지 않은 경우

---

### 3. ReentrantLock - 유연한 락

`java.util.concurrent.locks.ReentrantLock`은 `synchronized`의 한계를 극복한 락이다.

```java
public class ReentrantLockCounter {
    private final ReentrantLock lock = new ReentrantLock();
    private int count = 0;

    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock(); // 반드시 finally에서 해제
        }
    }

    public int getCount() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}
```

#### synchronized와의 차이

| 기능 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 사용법 | 키워드 (암묵적 획득/해제) | API 호출 (명시적 lock/unlock) |
| 대기 중 인터럽트 | 불가 | `lockInterruptibly()` |
| 타임아웃 | 불가 | `tryLock(timeout, unit)` |
| 공정성(Fairness) | 보장 안 됨 | `new ReentrantLock(true)` |
| Condition | `wait()/notify()` | `newCondition()` |
| 락 상태 확인 | 불가 | `isLocked()`, `isHeldByCurrentThread()` |

#### tryLock - 락 획득 시도와 타임아웃

```java
public boolean tryIncrement() {
    // 락 획득을 시도하되, 최대 1초만 대기
    boolean acquired = false;
    try {
        acquired = lock.tryLock(1, TimeUnit.SECONDS);
        if (acquired) {
            count++;
            return true;
        } else {
            // 1초 내에 락을 획득하지 못함 → 다른 처리
            System.out.println("락 획득 실패, 다른 로직 수행");
            return false;
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        return false;
    } finally {
        if (acquired) {
            lock.unlock();
        }
    }
}
```

`synchronized`에서는 불가능한 패턴이다. 락을 획득하지 못하면 **무한 대기**하는 수밖에 없지만, `tryLock`은 **타임아웃** 후 다른 처리를 할 수 있다. 데드락을 방지하는 데 유용하다.

#### 공정성(Fairness)

```java
// 공정한 락: 먼저 요청한 스레드가 먼저 획득 (FIFO)
ReentrantLock fairLock = new ReentrantLock(true);

// 불공정한 락(기본값): 성능은 좋지만 기아 발생 가능
ReentrantLock unfairLock = new ReentrantLock(false);
```

공정한 락은 **기아(Starvation)**를 방지한다. 다만 스레드 간 순서를 관리하는 오버헤드 때문에 처리량이 떨어질 수 있다. 대부분의 상황에서는 불공정 락(기본값)의 성능이 더 좋다.

#### Reentrant(재진입)이란?

이름에 "Reentrant"가 붙은 이유가 있다. **같은 스레드가 이미 획득한 락을 다시 획득할 수 있다.**

```java
public class ReentrantExample {
    private final ReentrantLock lock = new ReentrantLock();

    public void outer() {
        lock.lock();
        try {
            System.out.println("outer 진입");
            inner(); // 같은 락을 다시 획득 → 데드락 안 남
        } finally {
            lock.unlock();
        }
    }

    public void inner() {
        lock.lock(); // 같은 스레드이므로 재진입 가능
        try {
            System.out.println("inner 진입");
        } finally {
            lock.unlock();
        }
    }
}
```

재진입이 불가능한 락이었다면, `outer()`에서 락을 잡고 `inner()`를 호출할 때 **자기 자신이 잡은 락 때문에 데드락**이 발생한다. `synchronized`도 재진입을 지원한다.

---

### 4. ReadWriteLock - 읽기와 쓰기 분리

"읽기는 동시에 해도 되지만, 쓰기는 독점해야 한다"는 요구사항은 매우 흔하다. `ReadWriteLock`은 이 패턴을 위한 락이다.

```java
public class ReadWriteCache<K, V> {
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();
    private final Map<K, V> cache = new HashMap<>();

    public V get(K key) {
        readLock.lock();
        try {
            return cache.get(key);
        } finally {
            readLock.unlock();
        }
    }

    public void put(K key, V value) {
        writeLock.lock();
        try {
            cache.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }

    public int size() {
        readLock.lock();
        try {
            return cache.size();
        } finally {
            readLock.unlock();
        }
    }
}
```

#### 동작 규칙

| | 읽기 락 보유 중 | 쓰기 락 보유 중 |
|---|---|---|
| **읽기 락 요청** | 허용 (동시 읽기) | 대기 |
| **쓰기 락 요청** | 대기 | 대기 |

읽기가 압도적으로 많고 쓰기가 드문 상황(캐시, 설정값 관리 등)에서 `synchronized`보다 훨씬 높은 처리량을 보여준다.

---

## CAS(Compare-And-Swap) 원리와 Lock-Free 프로그래밍

### 락의 비용

지금까지 살펴본 `synchronized`와 `ReentrantLock`은 모두 **블로킹 방식**이다. 락을 획득하지 못한 스레드는 대기 상태에 빠진다. 이 대기에는 비용이 따른다.

```
락 기반 동기화의 비용:
1. 컨텍스트 스위칭 - 스레드 대기/깨우기에 드는 OS 비용
2. 스케줄링 지연 - 깨어난 스레드가 CPU를 다시 할당받을 때까지의 지연
3. 메모리 동기화 - 락 획득/해제 시 메모리 배리어 비용
4. 확장성 한계 - 스레드가 많을수록 경쟁이 심해지고 처리량 저하
```

경쟁이 심하지 않은 상황에서는 이 비용이 과도할 수 있다. **"누군가 이미 쓰고 있으면 기다리는 게 아니라, 다시 시도하면 안 될까?"** 이 발상이 CAS의 출발점이다.

### CAS란?

**CAS(Compare-And-Swap)**는 CPU 수준에서 제공하는 **원자적 연산**이다. 세 개의 인자를 받는다.

```
CAS(메모리 위치, 기대값, 새 값)

1. 메모리 위치의 현재 값이 기대값과 같으면 → 새 값으로 교체하고 true 반환
2. 메모리 위치의 현재 값이 기대값과 다르면 → 아무것도 하지 않고 false 반환
```

이 전체 과정이 **하나의 CPU 명령어**로 실행된다. 중간에 다른 스레드가 끼어들 수 없다.

```
시간  스레드A                         스레드B                         count
──────────────────────────────────────────────────────────────────────────
t1   READ count = 0                                                   0
t2                                   READ count = 0                   0
t3   CAS(count, 0, 1) → 성공!                                         1
t4                                   CAS(count, 0, 1) → 실패!         1
t5                                   READ count = 1 (재시도)           1
t6                                   CAS(count, 1, 2) → 성공!         2
```

스레드B는 CAS가 실패하면 **현재 값을 다시 읽고 재시도**한다. 이 패턴을 **스핀(Spin)**이라 한다. 락을 잡고 대기하는 대신, **반복 시도**로 동시성을 제어한다.

### Java에서의 CAS - Atomic 클래스

Java는 `java.util.concurrent.atomic` 패키지에서 CAS 기반 클래스들을 제공한다. 내부적으로 `Unsafe` 클래스(현재는 `VarHandle`)를 통해 CPU의 CAS 명령어를 직접 호출한다.

```java
public class AtomicCounter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet(); // CAS 기반 원자적 증가
    }

    public void decrement() {
        count.decrementAndGet();
    }

    public int getCount() {
        return count.get();
    }
}
```

#### AtomicInteger의 내부 동작

`incrementAndGet()`의 내부를 들여다보면 CAS 스핀 루프가 보인다.

```java
// AtomicInteger.incrementAndGet()의 개념적 구현
public int incrementAndGet() {
    while (true) {  // 스핀 루프
        int current = get();            // 현재 값 읽기
        int next = current + 1;         // 새 값 계산
        if (compareAndSet(current, next)) {  // CAS 시도
            return next;                // 성공하면 반환
        }
        // 실패하면 다시 시도 (다른 스레드가 값을 변경했으므로)
    }
}
```

락을 사용하지 않기 때문에 **Lock-Free**라고 부른다. 스레드가 대기 상태에 빠지지 않으므로 컨텍스트 스위칭 비용이 없다.

#### 주요 Atomic 클래스들

```java
// 정수형
AtomicInteger atomicInt = new AtomicInteger(0);
AtomicLong atomicLong = new AtomicLong(0L);

// 참조형
AtomicReference<String> atomicRef = new AtomicReference<>("initial");

// 배열
AtomicIntegerArray atomicArray = new AtomicIntegerArray(10);

// 필드 업데이터 (기존 클래스의 필드를 원자적으로 변경)
AtomicIntegerFieldUpdater<MyClass> updater =
    AtomicIntegerFieldUpdater.newUpdater(MyClass.class, "count");
```

### CAS 활용 예제: Lock-Free 스택

CAS를 사용해 락 없이 스레드 안전한 스택을 구현할 수 있다.

```java
public class LockFreeStack<T> {
    private final AtomicReference<Node<T>> top = new AtomicReference<>(null);

    private static class Node<T> {
        final T value;
        final Node<T> next;

        Node(T value, Node<T> next) {
            this.value = value;
            this.next = next;
        }
    }

    public void push(T value) {
        while (true) {
            Node<T> currentTop = top.get();
            Node<T> newTop = new Node<>(value, currentTop);
            if (top.compareAndSet(currentTop, newTop)) {
                return; // CAS 성공 → push 완료
            }
            // CAS 실패 → 다른 스레드가 top을 변경함 → 재시도
        }
    }

    public T pop() {
        while (true) {
            Node<T> currentTop = top.get();
            if (currentTop == null) {
                return null; // 스택이 비어있음
            }
            Node<T> newTop = currentTop.next;
            if (top.compareAndSet(currentTop, newTop)) {
                return currentTop.value; // CAS 성공 → pop 완료
            }
            // CAS 실패 → 재시도
        }
    }
}
```

`synchronized`나 `ReentrantLock` 없이도 스레드 안전한 스택이 완성되었다. 경쟁이 심하지 않다면 락 기반보다 훨씬 높은 처리량을 보여준다.

### ABA 문제

CAS에도 약점이 있다. **ABA 문제**다.

```
시간  스레드A              스레드B              값
──────────────────────────────────────────────────
t1   READ value = A                            A
t2   (일시 중단됨)         value = B로 변경      B
t3                        value = A로 변경      A
t4   CAS(A, A, C) → 성공!                      C
```

스레드A는 값이 A인 것을 확인하고 CAS를 시도한다. 하지만 그 사이에 값이 A → B → A로 변경되었다. CAS는 "현재 값 = 기대값"만 비교하므로 **중간에 변경이 있었는지 알 수 없다.**

대부분의 정수 카운터에서는 ABA가 문제되지 않는다. 하지만 **참조형(포인터)**에서는 심각한 버그로 이어질 수 있다. 예를 들어 위의 Lock-Free 스택에서, 노드가 재사용되면 pop된 노드가 다시 push되었을 때 ABA가 발생할 수 있다.

#### 해결: AtomicStampedReference

```java
AtomicStampedReference<String> stampedRef =
    new AtomicStampedReference<>("A", 0); // (값, 스탬프)

// 값뿐 아니라 스탬프(버전)까지 함께 비교
int[] stampHolder = new int[1];
String current = stampedRef.get(stampHolder);
int currentStamp = stampHolder[0];

// 값이 같더라도 스탬프가 다르면 CAS 실패
stampedRef.compareAndSet(
    current, "C",           // 기대값 → 새 값
    currentStamp, currentStamp + 1  // 기대 스탬프 → 새 스탬프
);
```

스탬프(버전 번호)를 함께 관리하면 중간에 변경이 있었는지 감지할 수 있다.

### CAS vs Lock, 언제 무엇을 쓸까?

| 기준 | CAS (Atomic) | Lock (synchronized/ReentrantLock) |
|------|-------------|-----------------------------------|
| **경쟁 빈도 낮음** | 유리 (스핀 비용 적음) | 불필요한 오버헤드 |
| **경쟁 빈도 높음** | 불리 (스핀 낭비 심함) | 유리 (대기 상태로 전환) |
| **연산 복잡도** | 단순 연산에 적합 | 복합 연산에 적합 |
| **임계 영역 크기** | 작을수록 유리 | 클수록 상대적 유리 |
| **데드락 위험** | 없음 | 있음 (잘못된 순서로 락 획득 시) |

핵심은 **경쟁 강도**다. CAS는 경쟁이 낮을 때 빛나고, 경쟁이 심할 때는 스핀에 CPU를 낭비한다. Lock은 경쟁이 심할 때 스레드를 대기시켜 CPU 자원을 아낀다.

---

## 동시성 제어 도구 정리

지금까지 살펴본 도구들을 정리하면 다음과 같다.

| 도구 | 핵심 용도 | 상호 배제 | 가시성 보장 | 블로킹 여부 |
|------|----------|----------|-----------|-----------|
| `synchronized` | 기본적인 동기화 | O | O | 블로킹 |
| `volatile` | 플래그 변수, 가시성 | X | O | 논블로킹 |
| `ReentrantLock` | 유연한 락 제어 | O | O | 블로킹 (tryLock은 논블로킹) |
| `ReadWriteLock` | 읽기-쓰기 분리 | O (쓰기만) | O | 블로킹 |
| `Atomic*` | 단순 연산의 원자성 | CAS 기반 | O | 논블로킹 (스핀) |

---

## 마무리

동시성 문제는 "발생할 수 있다"가 아니라, 멀티스레드 환경에서는 **"반드시 발생한다"**고 생각해야 한다. 문제는 항상 재현되지 않기 때문에, 테스트에서 통과했다고 안심하면 안 된다. 운영 환경에서 트래픽이 몰릴 때 비로소 드러나는 것이 동시성 버그의 무서운 점이다.

이번 글에서 다룬 내용을 요약하면 다음과 같다.

> **동시성 문제의 세 가지 원인**
> - 경쟁 조건(Race Condition): 공유 자원에 대한 비원자적 복합 연산
> - 가시성(Visibility): CPU 캐시로 인한 변경 전파 지연
> - 명령어 재배치(Reordering): JVM/CPU 최적화로 인한 실행 순서 변경

> **Java의 동시성 제어 도구**
> - `synchronized`: 간단하지만 유연성 부족
> - `volatile`: 가시성만 필요할 때
> - `ReentrantLock`: 타임아웃, 공정성, 인터럽트가 필요할 때
> - `ReadWriteLock`: 읽기 위주 워크로드
> - `Atomic*` 클래스: 단순 연산의 Lock-Free 처리

> **CAS의 핵심**
> - CPU 수준의 원자적 비교-교환 연산
> - 락 없이 동시성 제어 (Lock-Free)
> - 경쟁이 낮을 때 유리, 높을 때는 락이 유리

다음 편에서는 **DB Lock**으로 영역을 확장한다. 낙관적 락, 비관적 락, 그리고 Redis를 활용한 분산 락까지 - 애플리케이션 수준을 넘어서는 동시성 제어를 다룰 예정이다.

---

## 참고자료

- Brian Goetz 외, 「Java Concurrency in Practice」
- Doug Lea, 「Concurrent Programming in Java」
- Oracle, [java.util.concurrent.atomic 패키지 문서](https://docs.oracle.com/javase/17/docs/api/java.base/java/util/concurrent/atomic/package-summary.html)
