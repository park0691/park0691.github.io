---
title: Future는 왜 Completable해 졌을까?
date: 2026-02-11 12:00 +0900
categories: Java
tags: asynchronous, future, completablefuture
---

이 글은 비동기 처리를 공부하던 중, '왜 하필 Completable이라고 명명했을까?'라는 순수한 호기심에서 시작되었습니다. 이해를 돕기 위한 상황 예시와 코드는 정리 과정에서 Claude, Gemini의 도움을 받아 작성했습니다.

## 비동기 처리, 이렇게까지 복잡해야 할까?

비동기 처리는 `@Async` 어노테이션 하나면 다 되는 줄 알았는데, 실무에서는 여러 API를 동시에 호출하고 그 결과를 조합해야 하는 상황이 많았다.

다음은 실무에서 비동기를 적용하며 고민했던 내용을 각색한 코드다.

```java
import java.util.List;
import java.util.concurrent.*;

public class MyPageService {

    private final ExecutorService executor = Executors.newFixedThreadPool(10);

    public MyPageResponse getMyPageData(String userId) {
        
        System.out.println("1. 데이터 조회 시작 (Main Thread: " + Thread.currentThread().getName() + ")");

        // 1. 유저 정보 조회 API 비동기 호출 (약 2초 소요 가정하면)
        Future<UserProfile> profileFuture = executor.submit(() -> {
            return fetchUserProfile(userId);
        });

        // 2. 주문 내역 조회 API 비동기 호출 (약 3초 소요 가정하면)
        Future<List<Order>> orderFuture = executor.submit(() -> {
            return fetchOrderHistory(userId);
        });

        UserProfile profile = null;
        List<Order> orders = null;

        try {
            System.out.println("2. API 응답 대기 중... 메인 스레드는 여기서 멈춥니다(Blocking).");
            
            // [치명적 단점 1] 결과를 꺼내려면 무조건 기다려야 함 (Blocking)
            profile = profileFuture.get(); // 2초간 메인 스레드 정지
            orders = orderFuture.get();    // (위에서 2초 지났으니) 추가로 1초 더 정지

            System.out.println("3. 모든 데이터 수신 완료! 데이터 조합 시작.");

        } catch (InterruptedException e) {
            // [치명적 단점 2] 스레드 인터럽트 예외 처리의 번거로움
            Thread.currentThread().interrupt();
            throw new RuntimeException("작업이 중단되었습니다.", e);
        } catch (ExecutionException e) {
            // [치명적 단점 3] 비동기 작업 내부의 에러를 밖에서 까봐야 앎
            throw new RuntimeException("API 호출 중 에러가 발생했습니다.", e.getCause());
        }

        // 4. 두 데이터를 조합하여 최종 결과 반환
        return new MyPageResponse(profile, orders);
    }
    ...
}
```

처음에는 `Future`를 사용해 문제를 해결하려 했다. 하지만 코드는 `get()`과 `try-catch`로 도배되었고, 이게 과연 좋은 코드일까? 라는 의문이 들었다.

위 코드의 3가지 한계점은 다음과 같다.

1. **데이터 전달 방식은 블로킹 (Blocking `get()`)**<br/>
스레드 2개를 시켜서 일을 동시에 병렬로 시작하지만, 결국 결과를 조합하려면 메인 스레드가 `profileFuture.get()`과 `orderFuture.get()`을 호출해야 한다. 이 때 메인 스레드는 백그라운드 작업이 끝날 때까지 아무것도 하지 못하고 그 자리에 얼어붙게 된다(`Blocking`). **<u>`Future` 블럭 안 코드는 비동기로 실행되지만, 결과를 받아서 처리하는 코드는 결국 동기적으로 동작하는 셈이다.</u>**

2. **지저분한 예외 처리 (Exception Hell)**<br/>
`Future.get()`은 Checked Exception을 던진다. 비즈니스 로직보다 `try-catch` 블록이 차지하는 비중이 더 커져서 코드의 가독성이 떨어진다. 만약 둘 중 하나의 API라도 실패했을 때 '기본값(Default)을 세팅한다' 같은 복구 로직을 넣으려면 코드는 걷잡을 수 없이 지저분해진다.

3. **'이거 끝나면 저거 해!'가 안 됨 (콜백의 부재)**<br/>
가장 답답한 부분이다. `Future`는 **'결과가 올 때까지 기다린다'**만 될 뿐, **'주문 내역 조회가 완료되는 순간(Event), 그 데이터를 가지고 로그를 남겨라'**와 같은 후속 작업(Callback)을 예약할 수 없다. 개발자가 직접 루프를 돌며 `isDone()`을 체크하거나 `get()`으로 멈춰 서서 감시하는 수밖에 없다.

`CompletableFuture`의 주요 기능은 다음에 정리해두었다.

[https://park0691.github.io/TIL/java/thread-completable-future.html](https://park0691.github.io/TIL/java/thread-completable-future.html)

Java 8의 `CompletableFuture`를 파고들면서 비동기 프로그래밍 철학이 어떻게 바뀌었는지 알 수 있게 되었다. 이 포스트에서는 단순한 메서드 기능 나열이 아닌, 주니어의 관점에서 '왜 이름이 하필 `Completable-Future`인지', 그리고 과거의 기술들이 어떤 한계를 극복하며 지금의 모습으로 진화했는지 그 맥락을 짚어보고자 한다.


## 1. 비동기의 태동: `Runnable`과 공유 메모리 지옥

가장 원초적인 비동기 처리는 `new Thread(Runnable).start()` 였다. 하지만 `Runnable`의 `run()` 메서드 반환 타입은 `void`다. 스레드에게 일을 시킬 수는 있었지만 만들어낸 결과를 던져준 스레드(메인 스레드)가 직접 돌려받을 우아한 방법이 없었다.

결과를 공유하려면 아래 코드처럼 공용 메모리를 사용해야 했고, 이는 필연적으로 동기화(`synchronization`) 지옥을 낳았다.

```java
public class LegacyThreadExample {

    // 두 스레드가 결과를 주고받기 위한 '공용 메모리'
    private static int sharedResult = 0; 
    private static boolean isDone = false;
    private static final Object lock = new Object(); // 동기화를 위한 자물쇠

    public static void main(String[] args) throws InterruptedException {
        System.out.println("메인 스레드: 일꾼에게 작업을 지시합니다.");

        Thread worker = new Thread(() -> {
            // 1. 비즈니스 로직 수행 (예: 2초가 걸리는 무거운 계산)
            try { Thread.sleep(2000); } catch (InterruptedException e) {}

            // 2. 결과를 돌려줄(return) 방법이 없으니, 공용 메모리에 직접 쓴다.
            synchronized (lock) {
                sharedResult = 42; 
                isDone = true;
                lock.notify(); // "나 일 다 했어!" 하고 메인 스레드를 깨움
            }
        });

        worker.start();

        // 3. 메인 스레드는 일꾼이 결과를 공용 메모리에 쓸 때까지 기다려야 한다.
        synchronized (lock) {
            while (!isDone) {
                lock.wait(); // 일꾼이 깨워줄 때까지 하염없이 대기 (Blocking)
            }
            System.out.println("메인 스레드: 일꾼이 남긴 결과를 확인했습니다 -> " + sharedResult);
        }
    }
}
```

고작 숫자 하나를 넘기기 위해 `wait()`, `notify()`를 남발해야 했고 데드락의 위험도 항상 존재했다.

## 2. 절반의 성공: `Callable`과 `Future`의 등장 (영수증의 한계)

이러한 답답함을 해결하기 위해 Java 5에서 결과값을 반환할 수 있는 `Callable`, 그 결과를 담는 상자 `Future`가 등장했다.

하지만 서두의 코드에서 보았듯, 치명적인 단점이 있었다. **데이터를 회수하는 방식이 본질적으로 '블로킹(Blocking)'이라는 것**이다. 메인 스레드가 빈 상자를 열어보기 위해 `future.get()`을 호출하는 순간, 데이터가 아직 없다면 그 자리에 꼼짝 못 하고 멈춰 서야 했다. 여러 작업의 결과를 조합하려면 코드가 기형적으로 변했다.

또 다른 근본적인 문제는, 메인 스레드나 제3의 스레드가 이 `Future` 객체에 직접 **"내가 결과를 알아왔으니 이걸로 끝내!"**라고 값을 주입할 방법이 전혀 없었다는 것이다. `Future` 인터페이스에는 상태를 강제로 조작할 수 있는 메서드가 아예 존재하지 않았다. `Future`는 스레드 풀에 던져준 작업이 스스로 `return` 하며 끝날 때까지 밖에서는 그저 다 끝났는지 확인(`isDone`)하거나 끝날 때까지 기다리는(`get`)  수동적인 '닫힌 상자'였다.

## 3. 유레카: 그래서 왜 Completable 인가?
`CompletableFuture`를 공부하며 가장 크게 얻은 인사이트는 이름 그 자체에 있었다.

***'완료시킬 수 있는(Completable)' Future.***

과거의 `Future`는 외부에서 개입할 방법이 없는 '읽기 전용(Read-Only)' 상태판이었다. 하지만 `CompletableFuture`는 개발자가 외부에서 직접 `complete(값)` 메서드를 호출해 **"이 작업은 이 값으로 끝났어!"**라고 상태를 강제로 '쓰기(Write)' 할 수 있는 주도권을 줬다. **개발자가 원하는 시점에 어디서든 직접 상자를 닫을 수 있게 된 것이다.**

이게 왜 중요할까? 언제 올지 모르는 외부 시스템(메시지 큐 등)의 이벤트를 기다리는 아래 코드를 보자.

```java
public class EventService {

    // 외부 메시지 큐(RabbitMQ, Kafka 등) 구독 클라이언트 (가정)
    private final MessageQueueClient messageQueue = new MessageQueueClient();

    public CompletableFuture<String> waitForPaymentEvent(String orderId) {
        // 1. 일단 텅 빈 상자(CompletableFuture)를 하나 만든다.
        CompletableFuture<String> future = new CompletableFuture<>();

        // 2. 외부 시스템에 콜백(이벤트 리스너)을 등록한다. (언제 응답이 올지 모른다!)
        messageQueue.subscribe(orderId, new MessageListener() {
            @Override
            public void onMessageReceived(String eventData) {
                System.out.println("외부에서 결제 완료 이벤트 도착!");
                
                // 3. 핵심 포인트: 외부 이벤트가 발생하는 순간, 
                // 개발자가 직접 상자에 결과값을 찔러 넣고 완료 처리해 버린다! (Write 주도권)
                future.complete(eventData);
            }
        });

        // 4. 결과가 채워지기를 기다리는 '빈 상자'를 즉시 반환한다.
        return future;
    }
}
```

만약 이 코드를 `Future`로 구현하려 했다면 지옥이 열렸을 것이다. `Future`는 오직 백그라운드 스레드의 `return`으로만 완료될 수 있기 때문에, 언제 호출될지 모르는 저런 `onMessageReceived` 콜백 함수 안에서는 Future에 결과값을 넘겨줄 방법이 아예 존재하지 않는다.

하지만 `CompletableFuture`는 달랐다.

- **이벤트 기반 프로그래밍과의 결합**: 웹소켓이나 메시지 큐(RabbitMQ 등)에서 언제 올지 모르는 응답을 기다릴 때, 응답이 오는 순간 콜백 함수에서 `future.complete(응답값)`을 찔러주면 된다. 개발자가 비어있는 퓨처 상자를 쥐고 있다가, '외부 API 응답이 도착했네? 그럼 이 비동기 작업은 이 값으로 끝난 걸로 칠게!'라며 원하는 시점에 `Future`의 생명주기를 직접 닫아버릴 수 있게 되었다.

- **수동적인 대기에서 능동적인 제어로**: 단순히 스레드의 작업이 끝나기만을 블로킹하며 기다리는 것에서 벗어났다. 이제 개발자는 비동기 작업의 생명주기(Lifecycle)에 개입하여, 외부의 이벤트나 조건이 충족되는 즉시 해당 작업을 직접 '완료' 시킬 수 있게 되었다.

이 '외부에서 수동으로 완료시킬 수 있다'는 작은 차이가 비동기 프로그래밍의 패러다임을 완전히 바꿨다. 메인 스레드는 더 이상 수동적인 대기자가 아니다. **개발자가 주도권을 쥐고 여러 비동기 작업의 완료 시점을 능동적으로 제어**할 수 있게 된 것이다.

## 4. 파이프라인의 완성: 콜백과 체이닝
능동적으로 완료시킬 수 있는 상자(`CompletableFuture`)가 생기자, Java는 여기에 함수형 프로그래밍 철학을 끼워 넣었다.

`CompletableFuture`는 기존 `Future`처럼 데이터를 만드는 동작을 비동기로 수행한다는 점은 같지만, **<u>완성된 데이터를 전달할 때는 대기(Blocking)가 아닌 '비동기적 파이프라인'으로 흘려보낸다.</u>** 자바스크립트의 `Promise`처럼 비동기 작업들을 레고 블록 조립하듯 연결(Chaining)할 수 있다.

- `thenApply()`: A 작업이 끝나면 그 결과로 B를 해라.
    ```java
    CompletableFuture.supplyAsync(() -> fetchUserId())
                     .thenApply(id -> fetchUserProfile(id)); // 결과를 받아 다음 작업 수행
    ```

- `allOf()`: A, B, C 작업이 모두 끝날 때까지 기다렸다가 다음으로 넘어가라.
    ```java
    CompletableFuture.allOf(futureA, futureB, futureC)
                     .thenRun(() -> System.out.println("모든 API 호출 완료!"));
    ```

**[서두의 답답했던 코드는 어떻게 개선할 수 있을까?]**<br/>

앞서 살펴본 `Future` 기반의 마이페이지 조회 코드를 `CompletableFuture`로 리팩토링하면 다음과 같다.

```java
public MyPageResponse getMyPageData(String userId) {
    // 1. 유저 정보와 주문 내역을 동시에 비동기로 호출하고, 결과를 조합(thenCombine)한다.
    CompletableFuture<MyPageResponse> futureResponse = 
        CompletableFuture.supplyAsync(() -> fetchUserProfile(userId), executor)
        .thenCombine(
            CompletableFuture.supplyAsync(() -> fetchOrderHistory(userId), executor),
            (profile, orders) -> new MyPageResponse(profile, orders) // 두 결과가 모두 도착하면 조합!
        )
        // 2. 둘 중 하나라도 에러가 발생하면, 기본값으로 복구(exceptionally)한다.
        .exceptionally(ex -> {
            System.err.println("API 호출 실패, 기본 객체를 반환합니다: " + ex.getMessage());
            return new MyPageResponse(new UserProfile("알 수 없음", "일반"), List.of());
        });

    // 지저분한 try-catch 없이 깔끔하게 최종 결과만 꺼내거나 콜백으로 넘긴다.
    return futureResponse.join(); 
}
```

블로킹(`get`)도 없고, 지저분한 예외 처리(`try-catch`)도 사라졌다.

**"이 두 API를 동시에 호출하고, 둘 다 도착하면 조합해 줘. 만약 에러 나면 이걸로 대체해"**라는 논리적인 흐름(파이프라인)만 남았다. 실행은 JVM과 스레드 풀이 알아서 논블로킹으로 처리한다.

## 5. 한눈에 보는 Future vs CompletableFuture

| 메소드      | Future (Java 5) | CompletableFuture (Java 8) |
| ----------- | ---------- | ---------- |
| **결과 수집**    | `get()`을 통한 **동기 블로킹** 대기 | `thenAccept()`, `thenApply()` 등을 활용한 **비동기 콜백** |
| **결과 조합**    | 스레드 수만큼 `get()` 호출 및 수동 조합 | `thenCombine()`, `allOf()` 등으로 체이닝하여 조합 | 
| **예외 처리**    | 지저분한 `try ~ catch` 블록 도배 | `exceptionally()`, `handle()`을 통한 파이프라인 내장 처리 |
| **메인 스레드**   | 응답이 올 때까지 **대기 (Blocking)** | 흐름만 설계해 두고 **대기하지 않음 (Non-blocking)** |


## 마무리하며: 명령형에서 선언형으로의 사고 전환
단순히 "속도가 빨라진다"를 넘어, `CompletableFuture`가 내게 준 가장 큰 수확은 **'사고방식의 전환'**이었다.

이전까지 비동기 코드는 'A 호출해, 기다려. 결과 나오면 B API 호출해'라는 절차적이고 명령형(Imperative)인 방식이었다. 하지만 `CompletableFuture`를 적재적소에 쓰면서, 'A 응답이 오면 B를 호출하는 파이프라인을 만들어 둘게'라는 **선언적이고 반응형(Reactive)인 사고**를 할 수 있게 되었다.

물론 체이닝이 길어지면 코드가 직관적으로 읽히지 않는다는 단점도 있고, 스레드 풀 관리를 잘못하면 오히려 시스템 성능을 저하시킬 수 있는 양날의 검이기도 하다. 최근에는 Java 21의 **가상 스레드(Virtual Thread)**가 등장하면서 비동기 처리의 패러다임이 또 한 번 격변하고 있다. 하지만 `CompletableFuture`가 자바 비동기 역사에서 보여준 '데이터 흐름 중심'의 설계 철학은 앞으로 어떤 새로운 기술이 나오더라도 반드시 짚고 넘어가야 할 중요한 이정표라고 생각한다.


## References
- https://medium.com/@hjna316/future%EC%99%80-completablefuture-java-%EB%B9%84%EB%8F%99%EA%B8%B0%EC%9D%98-%EC%A7%84%ED%99%94-ede7ed606dc2
- https://hongchangsub.com/java-completablefuture/
- Claude Sonet 4
- Gemini 3.1 Pro