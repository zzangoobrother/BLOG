# FCM 개선기 2

이전에 FCM 전송에서 쓰레드 사용이 과다하게 발생하는 현상을 개선하였다. 이번에는 더 나아가서 설계를 개선 하였다.

시작하기 전 제한사항은 kafka 같은 메시지 큐를 사용하지 않는다. 많은 알림 시스템 개선 사항에서 메시지 큐를 사용하지만 현재 운영 중인 서비스에서 메시지 큐 사용은 오버엔지니어링이고,
회사에서 메시지 큐를 운영할 만한 여력이 안된다. 그래서 순수하게 Java 만을 사용하여 개선할 예정이다.

고려사항을 알아보자.

### 고려사항
- 데이터 손실 방지 : 어떤 사황에서도 알림이 소실되면 안 된다. 알림이 지연되거나 순서가 틀려도 괜찮지만, 사라지면 안된다.
- 알림 중복 전송 방지
- 전송률 제한
- 재시도 방법

위 고려사항 중 전송률 제한은 구현하지 않겠다. 이유는 현재 개선기는 회사에서 고려한 사항인데 운영중인 서비스는 그만큼 트래픽 발생이 안되기 때문에 전송률 제한까지 고려하지 않아도 괜찮았다.
나머지 3가지 고려사항만 고려하여 개선하였다.

#### 데이터 손실 방지
메시지와 FCM 전송 성공 상태를 관리를 하나의 트랜잭션에서 관리하면 데이터 손실을 방지할 수 있다.

#### 알림 중복 전송 방지
FCM 전송을 하면 Firebase에서 전송 성공 여부에 대한 응답 code를 통해 알 수 있고, 이를 통해 알림 중복 전송을 제어할 수 있다.

#### 재시도 방법
Spring batch 또는 scheduler를 이용하여 재시도 처리를 합니다.

이제부터 코드를 통해 어떻게 개선 했는지 보겠다.

우선 개선하고자 하는 구조를 보자.

![fcm_architecture_1.png](img/fcm_architecture_1.png)

다중 파이프라인을 통해 확장해 하나의 Queue에 부하를 분산하였습니다. 그리고 single consumer를 구현하였습다.  
자세한 이야기는 코드를 통해 보겠습니다.

#### Queue
```java
@Component
public class Queue {
    private final BlockingDeque<Message> queue = new LinkedBlockingDeque<>();

    public void add(Message message) {
        queue.offer(message);
    }

    public Message get() {
        try {
            return queue.take();
        } catch (InterruptedException e) {
            log.info("exception");
        }

        return null;
    }

    public int size() {
        return queue.size();
    }

    public boolean isEmpty() {
        return queue.isEmpty();
    }
}
```

Queue를 Blocking Deque로 구현했다. 이유는 동시성 프로그래밍에서 스레드에 안전한 Queue이고, 특정 조건이 충족될 때까지 스레드를 일시 중지시키는 연산으로, 연산이 완료될 때까지 스레드를 대기 상태로 만든다.

#### AsyncMultiProcessor
```java
@Component
public class AsyncMultiProcessor {

    private final List<Queue> queues;
    private final List<ExecutorService> flatterExecutors;
    private final List<ExecutorService> leaderExecutors;
    private Consumer<Message> function;
    private final int queueCount = 3;
    private final ObjectProvider<Queue> objectProvider;

    public AsyncMultiProcessor(ObjectProvider<Queue> objectProvider) {
        this.queues = new ArrayList<>(queueCount);
        this.flatterExecutors = new ArrayList<>(queueCount);
        this.leaderExecutors = new ArrayList<>(queueCount);
        this.objectProvider = objectProvider;
        setup();
    }

    private void setup() {
        for (int i = 0; i < queueCount; i++) {
            Queue queue = objectProvider.getObject();
            queues.add(queue);

            ExecutorService leaderExecutor = Executors.newSingleThreadExecutor();
            leaderExecutors.add(leaderExecutor);
            flatterExecutors.add(Executors.newSingleThreadExecutor());

            CompletableFuture.runAsync(() -> leaderTask(queue, leaderExecutor));
        }
    }

    private void leaderTask(Queue queue, ExecutorService follower) {
        while (!Thread.currentThread().isInterrupted()) {
            Message message = queue.get();
            follower.execute(() -> function.accept(message));
        }
    }

    public void init(Consumer<Message> function) {
        this.function = function;
    }

    public void produce(Message message) {
        if (Objects.isNull(message)) {
            return;
        }
        
        int selectedQueue = ThreadLocalRandom.current().nextInt(queueCount);
        flatterExecutors.get(selectedQueue).execute(() -> queues.get(selectedQueue).add(message));
    }
}
```

setup()을 보자. 확장 하고자 하는 Queue의 개수 만큼 for()문을 돌려 Queue를 생성하는데 여기서 Queue는 스프링 컨테이너에서 관리되어 싱글톤으로 관리되는데 ObjectProvider를 사용하여 새로운 인스턴스를 생성한다.  
leaderExecutors는 consumer 쓰레드 list이다. flatterExecutors는 Queue 쓰레드 list 이다.

#### MultiFcmConsumer
```java
@RequiredArgsConstructor
@Component
public class MultiFcmConsumer {

    private final AsyncMultiProcessor asyncMultiProcessor;
    private final SendService sendService;

    @PostConstruct
    public void init() {
        asyncMultiProcessor.init(this::consumer);
    }

    private void consumer(Message message) {
        log.info("consumer message : {}", message.getId());
        if (!Objects.isNull(message)) {
            sendService.send(message);
        }
    }
}
```

consumer()를 스프링 실행할 때 asyncMultiProcessor.init()에 주입하므로 별도의 쓰레드에서 실행됩니다.

#### 
```java
@RequiredArgsConstructor
@Service
public class SendService {

    private final FcmSend fcmSend;
    private final DeviceRepository deviceRepository;
    private final MessageDeviceRepository messageDeviceRepository;

    @Transactional
    public void send(Message message) {
        // ...

        for (int i = 0; i < sendResponses.size(); i++) {
            MessageDevice messageDevice = messageDevices.get(i);

            SendResponse sendResponse = sendResponses.get(i);
            if (sendResponse.isSuccessful()) {
                messageDevice.completed();
                continue;
            }
            
            MessagingErrorCode messagingErrorCode = sendResponse.getException().getMessagingErrorCode();

            // 재시도 처리
            if (MessagingErrorCode.QUOTA_EXCEEDED == messagingErrorCode
                    || MessagingErrorCode.UNAVAILABLE == messagingErrorCode
                    || MessagingErrorCode.INTERNAL == messagingErrorCode) {
                messageDevice.waiting();
            }
            // 없는 토큰으로 실패
            else if (MessagingErrorCode.UNREGISTERED == messagingErrorCode
                    || MessagingErrorCode.INVALID_ARGUMENT == messagingErrorCode) {
                messageDevice.cancel();
            }
        }
    }
}
```
이 코드에서 중요한거는 응답 코드에 따른 재시도/취소 처리를 해야한다.  
해당 응답 코드는 <a href='https://firebase.google.com/docs/cloud-messaging/send-message?hl=ko#java' target='_blank' >링크</a>에서 확인할 수 있다.

### 마치며
아직 서비스에 많은 사용자가 없기에 최대한 운영 서비스 내부 자원을 활용하여 개선을 했다.  
그리고 아직 더 개선할 부분도 남아 있다. FCM 알림 성능 시간이 10초 내외로 나온다. 실 사용자에게 성능에 대한 영향을 주지 않기 때문에 추후 개선 사항으로 미뤄두고 있는데 이 부분도 개선해야 한다.  
다음 FCM 개선기는 성능 개선에 집중해서 돌아올거 같다.  

자세한 데모 프로젝트 내용은 아래 리포지토리에서 볼 수 있다.

<a href='https://github.com/zzangoobrother/study-project/tree/master/fcm-project' target='_blank' >Project</a>
