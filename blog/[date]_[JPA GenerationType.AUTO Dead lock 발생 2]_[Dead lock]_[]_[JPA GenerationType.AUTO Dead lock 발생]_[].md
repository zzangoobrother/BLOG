# JPA GenerationType.AUTO Dead lock 발생 2

지난 편에 이어서 서버에서 어떤 일이 일어나고 있는지 알아보자.

### 서버 환경
- 쓰레드 수 : 4개
- 커넥션 수 : 3개
- 하나의 요청에 필요한 커넥션 수 : 2개

### 정상 상황

![dead_lock_1.png](img/dead_lock_1.png)

- 요청을 받습니다.
- DB Connection pool 에서 Connection 요청해 받아옵니다.
- 받은 Connection을 이용해서 DB 작업을 진행합니다.
- 처리가 끝나면 응답하고, Connection을 pool에 돌려줍니다.

### 동시에 4개 요청이 들어오면

![dead_lock_2.png](img/dead_lock_2.png)

작업을 시작하기 위해 3개의 쓰레드에서 Connection pool을 요청해서 받아갔고, 3번째 쓰레드는 Connection을 얻기 위해서 대기합니다. 또는 예외를 발생시킵니다.

### 서버 환경
- 쓰레드 수 : 2개
- 커넥션 수 : 2개
- 요청 하나를 처리하는데 필요한 커넥션 수 : 2개

![dead_lock_3.png](img/dead_lock_3.png)

두 개의 모든 쓰레드는 작업을 완료하기 위해 Connection이 하나 더 필요한데 없기 때문에 대기합니다.
기본 설정인 30초가 지나면 그제야 TimeoutException을 발생시키면서 가지고 있던 Connection을 모두 pool에 돌려줍니다.

### 해결 방법
#### 방법 1
`pool size = Tn * (Cm - 1) + 1`
- Tn : 전체 쓰레드 수
- Cm : 하나의 Task에서 동시에 필요한 최대 Connection 수

HikariCP 위키에서는 공식대로 Maximum pool size를 설정하면 Dead lock을 피할 수 있습니다.
- 전체 쓰레드 수 : 2
- 하나의 쓰레드에서 동시에 필요한 최대 Connection 수 : 2

`pool size = 2 * (2 - 1) + 1 = 3`

2개의 쓰레드가 동시에 HikariCP에 Connection을 요청하고 2개의 Connection을 골고루 나눠 갔습니다. 그럼에도 1개의 Connection이 남았습니다.
1개의 Connection이 Dead lock을 피할 수 있게 해주는 Key Connection이 됩니다. 남은 1개의 Connection을 2개의 쓰레드가 번갈아 사용하면서 작업을 마칠 수 있게 됩니다.

![dead_lock_4.png](img/dead_lock_4.png)

- 2개의 요청이 동시에 들어와서 두 쓰레드 모두 각각 하나의 Connection을 가져갔고, 두 쓰레드에서 추가적으로 Connection 하나를 더 얻으려는 시도가 발생합니다.

![dead_lock_5.png](img/dead_lock_5.png)

- 우측에 쓰레드가 먼저 Connection을 받아 작업을 처리합니다.
- 좌측 쓰레드는 여분의 Connection이 생길 때까지 대기합니다.

![dead_lock_6.png](img/dead_lock_6.png)

- 2번째 Connection을 먼저 받은 쓰레드가 처리하고 키 생성이 끝나면 여분의 Connection 만 pool에 반납 후 나머지 작업을 진행합니다.

![dead_lock_7.png](img/dead_lock_7.png)

- 좌측 쓰레드가 여분의 Connection을 받아 요청을 처리합니다.

여분의 Connection 덕에 Dead Lock이 걸리는 상황을 막았습니다. 물론 쓰레드 성능과 갯수에 따라서 Timeout이 날 가능성은 여전히 존해합니다.

#### 방법 2
ID 생성 전략을 바꾸는 것이 방법이 될 수 있습니다. 문제가 되는 키 전략이었던 TABLE의 경우엔 성능 자체도 그렇게 좋은 편이 아닙니다.
