# JPA GenerationType.AUTO Dead lock 발생 2

지난 편에 이어서 서버에서 어떤 일이 일어나고 있는지 알아보자.

### 서버 환경
- 쓰레드 수 : 4개
- 커넥션 수 : 3개
- 하나의 요청에 필요한 커넥션 수 : 2개

### 정상 상황

![dead_lock_1.png](img/dead_lock_1.png)


