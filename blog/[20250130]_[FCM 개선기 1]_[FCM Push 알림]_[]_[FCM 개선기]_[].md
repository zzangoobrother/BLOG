# FCM 개선기

### 현재 상황
회사에서 모니터링 중 FCM 알림 중 쓰레드 수가 너무 많이 생기는 이상 현상을 발견했다. 유사하게 코드를 작성 후 테스트를 통해 어떻게 나오는지 확인해 보자.

전체코드 : <a href='https://github.com/zzangoobrother/study-project/tree/master/fcm-project/src/main/java/com/example' target='_blank' >Github</a>

코드는 중요 부분만 하겠다.

```java
public record FcmMulticastMessage(
        Notification notification,
        Map<String, String> options,
        List<String> token
) {

    @Builder
    public FcmMulticastMessage {}

    public record Notification(
            String title,
            String body,
            String image
    ) {
        @Builder
        public Notification {}
    }
}

@Component
public class FcmClient {
    private static final String AUTH_URL = "firebase-auth-url";
    private FirebaseApp firebaseApp;

    @PostConstruct
    void getFirebaseAccessToken() {
        try {
            FirebaseOptions options = new FirebaseOptions.Builder()
                    .setCredentials(
                            GoogleCredentials
                                    .fromStream(new ClassPathResource(fcmProperties.serviceAccountFile()).getInputStream())
                                    .createScoped(Arrays.asList(AUTH_URL))
                    ).build();

            firebaseApp = FirebaseApp.initializeApp(options, "before");
        } catch (IOException e) {
            throw new RuntimeException(e.getMessage());
        }
    }

    // 1 : N 푸쉬 전송
    public void send(FcmMulticastMessage fcmMulticastMessage) {
        List<String> tokens = fcmMulticastMessage.token();
        List<List<String>> tokenPartition = Lists.partition(tokens, tokens.size() / 1000 + 1);
        List<MulticastMessage> multicastMessages = tokenPartition.stream()
                .map(it -> createMulticastMessage(it, fcmMulticastMessage.notification().title(), fcmMulticastMessage.notification().body(), fcmMulticastMessage.notification().image(), fcmMulticastMessage.options()))
                .toList();

        multicastMessages.forEach(it -> FirebaseMessaging.getInstance(firebaseApp).sendEachForMulticastAsync(it));
    }

    private MulticastMessage createMulticastMessage(List<String> tokens, String title, String content, String image, Map<String, String> options) {
        MulticastMessage message = MulticastMessage.builder()
                .setNotification(Notification.builder()
                        .setTitle(title)
                        .setBody(content)
                        .setImage(image)
                        .build())
                .addAllTokens(tokens)
                .build();

        if (!CollectionUtils.isEmpty(options)) {
            message = MulticastMessage.builder()
                    .putAllData(options)
                    .setNotification(Notification.builder()
                            .setTitle(title)
                            .setBody(content)
                            .setImage(image)
                            .build())
                    .addAllTokens(tokens)
                    .build();
        }

        return message;
    }
}
```

Fcm 알림을 하기 위한 기본 설정이다. 여기서 봐야할 부분은 `List<List<String>> tokenPartition = Lists.partition(tokens, tokens.size() / 1000 + 1);` 이다.
수 많은 token을 1000개로 나눠 firebase에 전송한다는 생각을 가지고 구현을 하였다.

이제 Fcm 알림을 전송해보자. token 개수는 3000개를 전송한다.

```java
@Slf4j
@RestController
public class FcmTestController {
    @GetMapping("/api/v1/before/fcm/test")
    public void fcmTest() {
        Map<String, String> options = new HashMap<>();
        options.put("TYPE", "test");

        List<String> tokens = createdToken();
        log.info("token size : {}", tokens.size());
        fcmClient.send(FcmMulticastMessage.builder()
                .notification(FcmMulticastMessage.Notification.builder()
                        .title("test title")
                        .body("test content")
                        .build())
                .token(tokens)
                .options(options)
                .build());
    }

    private List<String> createdToken() {
        List<String> tokens = new ArrayList<>();
        for (int i = 0; i < 3000; i++) {
            tokens.add("test" + i);
        }

        return tokens;
    }
}
```

![postman_fcm_before.png](img/postman_fcm_before.png)

![console_fcm_before.png](img/console_fcm_before.png)

3000개의 token을 잘 전송했다. 이제 중요한 쓰레드 수를 보자. 쓰레드 수를 확인하기 위해 Actuator 를 활용하여 확인했다. 쓰레드 수의 peek를 확인 해봤다.

![thread_fcm_before.png](img/thread_fcm_before.png)

쓰레드 수가 3779개가 생성된다. token 수 3000개와 그 외 코드를 수행하는데 발생되는 쓰레드 수 이다.

기존 생각에는 1000개의 token을 묶어서 firebase에 전송하는줄 알았는데 쓰레드 1개에 1개의 token을 전송 했었다.

쓰레드 수를 개선 해보자. 그러기 위해 우선 왜 쓰레드가 tokne 수가 1:1 로 발생되는지 알아보자.

![fcm_debug_1.png](img/fcm_debug_1.png)

![fcm_debug_2.png](img/fcm_debug_2.png)

![fcm_debug_3.png](img/fcm_debug_3.png)

![fcm_debug_4.png](img/fcm_debug_4.png)

![fcm_debug_5.png](img/fcm_debug_5.png)

![fcm_debug_6.png](img/fcm_debug_6.png)

![fcm_debug_7.png](img/fcm_debug_7.png)

![fcm_debug_8.png](img/fcm_debug_8.png)

![fcm_debug_9.png](img/fcm_debug_9.png)

결론부터 말하자면 `Executors.newCachedThreadPool(threadFactory);`를 사용하는데, 요청이 100개면 100의 쓰레드가, 1000개면 1000개의 쓰레드가 생성되고, 사용한 후 일정 시간이 지나도록 사용되지 않으면 소멸된다.
왜 `newCachedThreadPool()`를 사용했는지 유추해보자면 순식간에 FCM 알림을 전송한다는 전략으로 한거라 생각된다. 하지만 쓰레드 생성과 소멸은 많은 비용이 발생한다.

firebase에서는 이 문제를 알고 있었고, 가이드 또한 주었다. <a href='https://firebase.google.com/docs/reference/admin/java/reference/com/google/firebase/ThreadManager' target='_blank' >가이드</a> 이 부분을 개선하자.

ThreadManager를 상속해 테스트하기 위해 간단하게 구현했다.

```java
public class CustomThreadManager extends ThreadManager {
    @Override
    protected ExecutorService getExecutor(FirebaseApp firebaseApp) {
        return Executors.newFixedThreadPool(50);
    }

    @Override
    protected void releaseExecutor(FirebaseApp firebaseApp, ExecutorService executorService) {
        executorService.shutdownNow();
    }

    @Override
    protected ThreadFactory getThreadFactory() {
        return Executors.defaultThreadFactory();
    }
}
```

`newCachedThreadPool` 대신 `newFixedThreadPool` 을 사용한 이유는 `maxThread`를 설정하여 무한정 쓰레드 생성을 막을 수 있기 때문이다.
그리고 쓰레드의 개수는 사용자와 서버의 사양에 따라 설정하면 된다.

이제 만들었으니 적용해보자. 기존 firebase 설정 코드에 `setThreadManager()` 을 추가하기만 하면된다.

```java
@PostConstruct
void getFirebaseAccessToken() {
    try {
        FirebaseOptions options = new FirebaseOptions.Builder()
                .setCredentials(
                        GoogleCredentials
                                .fromStream(new ClassPathResource(fcmProperties.serviceAccountFile()).getInputStream())
                                .createScoped(Arrays.asList(AUTH_URL))
                )
                .setThreadManager(new CustomThreadManager())
                .build();
        firebaseApp = FirebaseApp.initializeApp(options, "after");
        
        this.firebaseMessaging = FirebaseMessaging.getInstance(firebaseApp);
    } catch (IOException e) {
        throw new RuntimeException(e.getMessage());
    }
}
```

설정도 완료 했으니 다시 테스트 해보자.

![postman_fcm_after.png](img/postman_fcm_after.png)

![console_fcm_after.png](img/console_fcm_after.png)

![thread_fcm_after.png](img/thread_fcm_after.png)

3000개의 token을 잘 전송되고, 쓰레드 수 또한 50개로 잘 동작한다.

여기까지 쓰레드 개선기 였다. 하지만 아직 부족하다. 다음편에서 조그 더 개선해보자.
