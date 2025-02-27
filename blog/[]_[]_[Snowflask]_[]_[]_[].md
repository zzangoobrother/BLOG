# 분산 시스템을 위한 유일 ID 생성기 설계

### 개요
보통 개발을 하면서 auto_increment를 사용하여 유일한 ID를 만들어 사용합니다. 보통 RDB에서 제공하는 auto_increment 기능을 사용합니다.  
하지만 대용량 데이터, 대용량 트래픽이 들어온다면 적절하지 않을 겁니다. 그리고 ID 채번 방식이 순서대로 채번을 하기 때문에 보안에도 취약합니다.

#### 장점
- ID 생성에 대해 따로 고민하지 않아도 됩니다. 편하게 생성이 가능합니다.
- DBMS에서 자동으로 최적화를 해줍니다.
- 정렬된 숫자 형식으로 순서가 보장되며 ID 기준으로 정렬 가능하다.

#### 단점
- 유일성을 보장하기 위해 1대의 DB에서만 ID 채번을 진행한다. 따라서 스케일 아웃이 필요할 때 어려움을 겪을 수 있다.
- ID 생성을 DB에 의존해야 합니다. insert가 실행된 이후에 PK를 알 수 있습니다. 즉, DB에 강한 의존이 생기게 되며 DB 서버를 한 대만 사용한다면 SOPF가 될 수 있습니다.

그렇다면 장점을 수용하고, 단점을 극복하는 ID 생성은 어떻게 할 수 있을까?

### 분산 시스템 ID 생성 방법

분산 시슽ㅁ에서 유일성이 보장되는 ID를 만드는 방법은 여러 가지가 있다.

- <a href='https://en.wikipedia.org/wiki/Universally_unique_identifier' target='_blank' >UUID</a>
- <a href='https://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/' target='_blank' >Ticket server</a>
- <a href='https://github.com/ai/nanoid' target='_blank' >Nano ID</a>
- <a href='https://github.com/twitter-archive/snowflake/tree/snowflake-2010' target='_blank' >Snowflake ID</a>

### UUID
UUID는 유일성이 보장되는 128비트의 16진수로 이루어진 숫자이다. 분산 컴퓨팅 환경에서 고유 식별자뿐만 아니라, 트랜잭션 ID, URI 등 고유한 값을 생성할 때 자주 사용됩니다.  
UUID는 중복될 가능성이 매우 낮습니다. 동일한 UUID가 생성될 확률을 50%로 끌어올리기 위해서는 초당 10억 개의 UUID를 100년 동안 계속해서 만들어야 합니다.

![snowflake_uuid.png](img/snowflake_uuid.png)

#### 장점
- 분산 시스템, 멀티 인스턴스 등 다양한 환경에서 유일한 ID로 활용 가능하다.
- UUID를 만드는 것은 단순하다. 그래서 쉽게 사용 가능하다.

#### 단점
- 128비트 이므로 크기가 크며, 데이터베이스 인덱스로 활용 시 공간 효율성이 좋지 않다.
- 읽고 이해하기 힘들다.
- 시간순으로 정렬할 수 없다.

### Ticket server
auto_increment 기능을 가진 DBMS를 Ticket server를 중앙 집중형으로 하나만 사용하는 것이다.

![snowflake_ticket_server.png](img/snowflake_ticket_server.png)

#### 장점
- 유일성이 보장되는 오직 숫자로만 구성된 ID를 쉽게 생성한다.
- 구현하기 쉽고, 중소 규모에 적합하다.
#### 단점
- Ticket server가 SPOF가 된다. 이를 피하기 위해 Ticket server를 여러 대 준비하면 데이터 동기화 같은 새로운 문제가 발생한다.

#### Nano ID
비교적 최근에 개발된 랜덤 기반 ID 라이브러리, UUID의 크기가 너무 커서 줄여주는 목적으로 사용한다.

UUID와 차이점
- 알파벳 대문자도 활용한다.
- 기본 21바이트이다, 가변 길이이다.

Maven, Gradle을 통해 추가해서 사용할 수 있습니다. <a href='https://github.com/aventrix/jnanoid' target='_blank' >링크</a>를 보면 SecureRandom을 사용해서 안전한 랜덤값 기반으로 ID를 생성할 수 있습니다.

### Snowflake ID
유일성이 보장되는 64bit의 10진수로 이루어진 숫자이다. 트위터(X)에서 많은 양의 데이터가 매일 생산되고 소비된다.  
매우 많은 트래픽을 처리하기 위해 분산 환경에서 서비스가 진행되는데, 각 데이터의 고유한 ID를 생성하기 위해 Snowflake ID 기법을 고안했다.

![snowflake_snowflakeId.png](img/snowflake_snowflakeId.png)

최 4개의 부분으로 구성되어 있다.

- sign(1bit) : 음수와 양수를 구별하는데 사용한다.
- timestamp(41bit) : 기원 시각(epoch) 이후로 몇 밀리초가 경과했는지 나타내는 값
- worker_id(10bit) : 데이터 센터 ID 5bit + 서버 ID 5bit, 데이터 센터 max 32개 * 서버 max 32대 => 1024대 까지 가능
- sequence(12bit) : ID를 생성할 때마다 1만큼 증가시키는 값이며, 1밀리 초가 경과할 때마다 0으로 초기화 된다.

#### 장점
- 64bit 크기로, GUID 표준에 비해 작은 크기를 가지고 있다.
- timestamp 기반으로 생성되기 때문에 정렬이 가능하다.

#### 단점
- timestamp 기반으로 생성되기 때문에 OS별, 인스턴스별 시간 차이가 발생할 수 있다.
- UUID에 비해 복잡하다.
- 중앙 집중 ID 채번 방식을 사용한다면 네트워크 통신이 필요, 관리 포인트가 늘어난다.

### 코드 작성
