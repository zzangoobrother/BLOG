# JPA GenerationType.AUTO Dead lock 발생 1

'DB Connection 갯수가 많다고 무조건 좋은건 아니다.' 라는 주제를 가지고 공부 중 재밌는 내용을 봤다.

JPA GenerationType.AUTO Dead lock이 발생하는 경우인데 가지고 있는 지식으로는 DB Connection 1개 가지고 동작을 하는게 아닌가? 왜 Dead lock 이 발생하지? 라는 생각이 들었다.

왜 Dead lock이 발생하는지 들어가 보자.

```java
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    private String name;

    private int age;

    public Member(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```

```java
@Transactional
public Member create(MemberRequest request) {
    return memberRepository.save(new Member(request.name(), request.age()));
}
```

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 1
```

HikariCP Connection을 1개로 하고 테스트를 해보자.
결과는 아래 이미지와 같다.

![스크린샷 2025-01-12 오후 4.35.42.png](img/스크린샷 2025-01-12 오후 4.35.42.png)

결과는 500 Internal Server Error 이고, DB Connection 기본 connection-timeout이 30초 때문에 30초 이상의 실행 결과가 나왔다.

이제 왜 이런 결과가 나왔는지 들어가보자

### 원인은 바로 @GeneratedValue(strategy = GenerationType.AUTO)
JPA에서 insert시 id 생성 방식을 결정하는 Annotation 입니다.
MySQL에서 hibernate_sequenct 테이블 단인 row를 사용하여 id를 생성한다.
hibernate_sequence 테이블 조회, update를 하면서 sub Transaction을 생성하여 실행한다.

정상 실행될 때의 sql 입니다.
```sql
// 1
select next_val as id_val from hibernate_sequence for update
// 2
update hibernate_sequence set next_val= ? where next_val=?
// 3
insert into member (age,name,id) values (?,?,?)
```

1번 sql를 보면 id 값의 중복 발생을 막기 위해 MySQL의 for update 쿼리를 사용하여 row lock를 걸어 다른 접근을 막는다.

### 실행 코드

#### SimpleJpaRepository
![스크린샷 2025-01-13 오후 9.04.51.png](img/스크린샷 2025-01-13 오후 9.04.51.png)

#### AbstractSaveEventListener.saveWithGeneratedId
![스크린샷 2025-01-13 오후 9.10.58.png](img/스크린샷 2025-01-13 오후 9.10.58.png)

#### SequenceStyleGenerator.generate
![스크린샷 2025-01-13 오후 9.13.39.png](img/스크린샷 2025-01-13 오후 9.13.39.png)

#### TableStructure.buildCallback
![스크린샷 2025-01-13 오후 9.15.59.png](img/스크린샷 2025-01-13 오후 9.15.59.png)

#### JdbcIsolationDelegate.delegateWork
![스크린샷 2025-01-13 오후 9.17.12.png](img/스크린샷 2025-01-13 오후 9.17.12.png)

ID 채번을 위해
`Connection connection = this.jdbcConnectionAccess().obtainConnection();`
코드가 실행 되며 2번째 Connection을 가지고 옵니다.

ID를 조회하고, update 하는 Transaction이 commit되면 Connection이 바로 Pool에 반납 됩니다.

지금까지 코드를 보면서 원인을 찾았습니다. 서버에서 어떤 상황들이 발생하는지 다음 편에서 알아보자!
