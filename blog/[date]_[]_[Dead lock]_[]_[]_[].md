# JPA GenerationType.AUTO Dead lock 발생

'DB Connection 갯수가 많다고 무조건 좋은건 아니다.' 라는 주제를 가지고 공부 중 재밌는 내용을 봤다.

JPA GenerationType.AUTO Dead lock이 발생하는 경우인데 가지고 있는 지식으로는 DB Connection 1개 가지고 동작을 하는게 아닌가? 왜 Dead lock 이 발생하지? 라는 생각이 들었다.

왜 Dead lock이 발생하는지 들어가 보자.


