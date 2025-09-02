---
title: Java 동시성 제어 기법 (2) - ReentrantLock
date: 2025-05-22 18:00 +0900
categories: Java
tags: reentrantlock, reentrant, lock, concurrent, synchronization, 동기화
---

이번 포스트에서는 `synchronized`보다 더 유연하게 동시성 제어를 할 수 있는 `ReentrantLock`에 대해 자세히 알아보고자 한다. `ReentrantLock`은 Java 5부터 도입된 concurrent 패키지의 `java.util.concurrent.locks.Lock` 인터페이스 구현체다. Lock 인터페이스에 대해 알아보고 ReentrantLock과 ReentrantReadWriteLock, StampedLock에 대해서도 간략히 알아볼 것이다.

## java.util.concurrent.locks.Lock
locks 패키지에 정의된 상호 배제를 위한 Lock API는 `synchronized`보다 더욱 유연하고 정교하게 동기화 처리를 할 수 있다.

> concurrent 패키지
> `java.util.concurrent`는 Java 5에서 추가된 패키지로 동기화와 관련된 다양한 유틸리티 클래스를 제공한다. 패키지에서 제공하는 주요 기능은 다음과 같다.
> 
> - locks : 상호 배제를 사용할 수 있는 클래스를 제공한다.
> - atomic : 동기화가 되어있는 변수를 제공한다.
> - executors : 스레드 풀 생성, 스레드 생명주기 관리, Task 등록과 실행 등을 간편하게 처리할 수 있다.
> - queue : thread-safe한 FIFO 큐를 제공한다.
> - synchronizers : 특수한 목적의 동기화를 처리하는 5개의 클래스를 제공한다. (Semaphore, CountDownLatch, CyclicBarrier, Phaser, Exchanger)
> 
{: .prompt-tip }

> concurrent.locks 패키지 주요 인터페이스
> - Lock : 공유 자원에 한번에 한 스레드만 read, write를 수행 가능하도록 한다.
> - ReadWriteLock : Lock에서 한 단계 발전된 메커니즘을 제공. 공유 자원에 여러 스레드가 읽을 수 있도록 락을 유지하고 쓸 때는 한 스레드만 접근 가능하게 한다.
> - Condition : Object 클래스의 monitor method인 wait, nofity, notifyAll 메서드를 대체한다. wait → await, notify → signal, notifyAll → signalAll로 생각하면 된다.
>
{: .prompt-warning }


### synchronized와의 차이
**[유연성]**

- 암묵적 락 해제 : `synchronized`는 블록 구조를 사용하기 때문에 하나의 블록 안에 임계 영역의 시작과 끝이 있어야 한다. 블록 `{}`이 끝나거나, 블록 안에서 예외가 발생하여 빠져나가면 JVM이 자동으로 락을 해제한다. 개발자가 락 해제를 신경 쓸 필요가 없다.

    ```java
    public void implicitUnlock() {
        synchronized(this) {
            // ...임계 영역...
        } // 이 지점에서 자동으로 unlock이 일어남
    }
    ```

- 명시적 락 해제 : Lock은 `lock()`으로 락을 직접 획득하고 반드시 `unlock()`을 호출하여 락을 수동으로 해제해야 한다. 따라서 임계 영역을 여러 메서드에 나눠서 작성할 수 있다.

    ```java
    Lock lock = new ReentrantLock();
    public void explicitUnlock() {
        lock.lock();
        try {
            // ...임계 영역...
        } finally {
            lock.unlock(); // 반드시 수동으로 unlock 해야 함
        }
    }
    ```

    - 이러한 유연성은 개발자에게 명시적으로 락을 해제해야 하는 책임을 부여하며, 적절한 `unlock()` 호출을 누락하면 데드락이나 리소스 누수로 이어질 수 있다.

**[공정성]**

- `synchronized`는 여러 스레드가 경쟁 상태에 있을 때 어떤 스레드가 다음 락을 획득할지 순서를 보장할 수 없다. (Non-fair) 락을 오래 기다린 스레드보다 이제 막 락을 요청한 스레드가 먼저 락을 획득하는 기아 상태 발생할 수 있다.
- 반면, Lock은 공정 모드 설정을 통해 락 획득 순서의 공정성을 보장한다.

**[세밀한 조건부 제어]**

- 대기 중인 스레드를 선별적으로 깨울 수 있다. (`Condition` 객체로 다중 조건 지원)
- `synchronized`는 JVM에 내장된 단일 조건 큐(Condition Queue) 만을 사용한다. `wait()`, `notify()`, `notifyAll()` 메서드로 스레드를 제어하는데, `notify()`는 어떤 스레드가 깨어날지 알 수 없고, `notifyAll()`는 불필요한 스레드까지 모드 깨워 성능 저하를 유발할 수 있다.
- `Lock`은 `newCondition()` 메서드를 통해 여러 개의 `Condition` 객체를 생성할 수 있다. 이를 통해 대기 중인 스레드를 목적에 따라 구분하여 관리하고, <u>원하는 조건의 스레드만 선별적으로 깨울 수 있어 정교한 제어가 가능</u>하다.

**[\[락 대기 중단\]](#lockinterruptibly)**
- `synchronized` 블록에 진입한 스레드는 다른 스레드가 락을 해제할 때까지 무한정 기다려야만 한다. `Thread.interrupt()`를 호출해도 대기 상태가 중단되지 않아, 교착 상태(Deadlock)에 빠진 스레드를 외부에서 중단시킬 방법이 없다.
- `Lock`은 `lockInterruptibly()` 메서드를 제공하여 락을 기다리는 도중에 `InterruptedException`을 발생시켜 대기를 중단하고 다른 작업을 수행하도록 제어할 수 있다.

**[락 상태 확인 및 제어 능력]**

- `synchronized`는 일단 락을 얻으려고 시도하면, 락이 풀릴 때까지 무조건 대기(blocking) 밖에 못한다.
- `ReentrantLock`은  스레드가 락을 기다리는동안 락을 제어하는 강력한 메서드를 제공한다.
    - 타임 아웃 지정 가능
    - 락 폴링 지원, Lock Polling
        - 락을 획득하기 위해 무작정 기다리는 대신, 주기적으로 락을 획득할 수 있는지 확인하는 기법. `tryLock()` 메서드로 구현한다.
        - `tryLock()`을 호출하는 순간 락을 획득할 수 있으면 `true`를 반환하며, 획득할 수 없다면 기다리지 않고 즉시 `false`를 반환한다. 이 `tryLock()` 메서드를 통해 다양한 락 획득 전략을 구사할 수 있다.
        
        ```java
        ReentrantLock lock = new ReentrantLock();

        // 스레드 A가 락을 오랫동안 점유하고 있다고 가정
        // lock.lock(); 

        // 스레드 B의 작업
        Runnable task = () -> {
            while (true) {
                // 1. 락 획득 시도 (폴링)
                if (lock.tryLock()) {
                    try {
                        System.out.println("드디어 락 획득 성공! 작업을 시작합니다.");
                        // 임계 영역: 실제 작업 수행
                        break; // 루프 탈출
                    } finally {
                        lock.unlock();
                    }
                } else {
                    // 2. 락 획득 실패 시
                    System.out.println("락 획득 실패. 다른 작업을 잠시 수행합니다.");
                    try {
                        // 락을 얻지 못한 동안 다른 유용한 작업을 하거나,
                        // CPU 낭비를 막기 위해 잠시 대기
                        Thread.sleep(500); 
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        };

        new Thread(task).start();
        ```
        
        - 장점
            - 스레드가 멈추지 않는다. `synchronized`를 사용했다면 스레드는 락이 풀릴 때까지 대기하지만, 락 폴링 기법을 통해 다른 대체 작업을 수행할 수 있어 시스템 응답성, 유연성이 크게 향상된다.

### Lock API
![lock](https://1drv.ms/i/c/9251ef56e0951664/IQS6uJN436iMQr_Ei5KGtSUvAdKYRxKIViYugkFfm-Nc0Ik?width=1024)
_출처 : https://javarevisited.substack.com/p/master-reentrantlock-in-java-with_


#### 주요 메소드

| 메소드 | 설명 |
| ------- | ------- |
| void lock()   | 락을 획득. 락 획득 실패한 경우 스레드는 대기<br/>인터럽트에 응답하지 않는다. |
| void lockInterruptibly()   | 현재 스레드가 `interrupted` 상태 아닐 때 락을 획득<br/>`lock()`과 달리 락 획득 대기 중 인터럽트 발생(`interrupt()`) 시 `InterruptedException` 발생 후 락 획득 포기 |
| boolean tryLock()   | 락 획득을 시도하고 즉시 결과 반환(lock polling)<br/>락을 선점한 스레드가 없을 때만 락을 얻으려고 시도<br/>성공 여부를 즉시 반환하므로 인터럽트에 응답 필요 없음 |
| boolean tryLock(long time, TimeUnit unit)    | `tryLock()`과 동일하지만 락 획득 실패한 경우 파라미터로 주어진 시간동안 락 획득 시도<br/>주어진 시간 동안 락을 얻지 못했더라도 대기 상태에 영원히 빠지지 않을 수 있다.<br/>대기 중 인터럽트 발생 시 `InterruptedException` 발생 후 락 획득 포기 |
| void unlock()   | 락 해제. 락을 획득한 스레드가 호출해야 함 |
| Condition newCondition()    | 현재 락과 연결된 `Condition` 객체 반환 |


#### 사용법
```java
class LockTest {
    private final Lock lock = new ReentrantLock();

    public void usage() {
        lock.lock();
        try {
            // Critical Section
            // do something...
        } finally {
            lock.unlock();
        }
    }
}
```
다른 범위에서 락 획득/해제가 필요한 경우 락이 유지되는동안 실행되는 모든 코드가 `try ~ catch, try ~ finally` 로 보호되어 필요 시 락이 해제되도록 해야 한다.

##### lockInterruptibly()
```java
Lock lock = new ReentrantLock();

@Override
public void run() {
    while (true) {
        try {
            // lock() : 락을 다른 스레드에서 사용하는 경우 다른 스레드가 unlock할 때까지
            // 이 스레드는 중단된다. 다른 스레드에서 interrupt() 호출해도 깨어나지 않는다.
            // lock.lock();
            lock.lockInterruptibly();	
            ...
        } catch (InterruptedException e) {
            System.out.println("대기 중 스레드가 인터럽트를 받았습니다.");
            System.out.println("락 대기를 중단합니다!")
        }
    }
}
```
`lockInterruptibly()` 메서드를 사용했을 때, 다른 스레드에서 `interrupt()`를 호출하면 해당 스레드는 깨어나 `InterruptedException`에 대한 catch 블록을 수행한다.

##### trylock()
`trylock()` 메서드를 사용하면 `lock()` 메서드처럼 락 객체를 얻는다. 락을 다른 스레드에서 사용 중인 경우 `lock()`은 해당 스레드를 중단한다. 반면, `tryLock()` 메서드는 `false`를 반환하고 다음 명령으로 넘어간다. 즉, 해당 스레드가 중단되지 않고 락 객체를 얻은 경우와 얻지 못한 경우로 나누어 다음 명령을 수행할 수도 있다.

```java
Lock lock = new ReentrantLock();

if (lock.tryLock()) {  // lock을 얻은 경우
    try {
        ...
    } finally {
        lock.unlock();
    }
} else {               // lock을 얻지 못한 경우
    ...
}
```
따라서 락을 즉시 얻을 수 있을 때만 작업을 진행하고, 획득 실패 시 다른 처리를 하고 싶을 때 유용하게 활용할 수 있다.

> `tryLock()` 메서드는 공정성을 따르지 않는다. 락 획득이 가능하다면 대기 중인 스레드를 무시하고 바로 선점할 수 있다. 만약 `tryLock()`을 사용하면서 공정성을 유지하고자 한다면 타임아웃을 인자로 받아 0초로 설정하면 된다. `tryLock(0, TimeUnit.SECONDS);`
{: .prompt-tip }

## ReentrantLock
재진입 가능한(한 스레드가 이미 확보한 락을 여러 번 걸 수 있음) 명시적 락으로, `synchronized`로 접근하는 모니터 락과 동일한 동작을 한다.

> 명시적 락<br/>
> 기존 `synchronized`는 암묵적 락으로 프로그래머가 직접 락을 제어하지 않아도 Java가 자동으로 락을 관리(획득/해제)해 줬다. 명시적 락은 프로그래머가 직접 락 객체를 만들고 `lock()`, `unlock()`을 직접 호출하여 락의 획득/해제를 명시적으로 수행하는 방식을 말한다.
> 
{: .prompt-tip }

동일한 스레드 내 최대 2^31 - 1(2,147,483,647) 번의 재귀적 락을 지원하며, 한도 초과시 메서드에서 예외를 발생시킨다.

### 재진입성
```java
public class SynchronizedTest {
    public synchronized outer() {
        inner();
    }
    
    public synchronized inner() {
        // do something...
    }
}
```
`synchronized` 블록은 재진입 가능하다. 자바 스레드가 `synchronized` 코드 블럭에 진입하면 동기화된 블럭의 모니터 락을 얻는다. 그 스레드는 다른 동일한 모니터 락을 얻는 동기화 코드 블럭에 진입할 수 있다.

즉, `outer()` 호출한 메서드가 해당 객체의 락을 얻고 `inner()` 메서드를 호출한다면 `inner()` 또한 `synchronized` 블록이기 때문에 락을 얻어야 하지만 해당 객체에 해당하는 락을 얻었기 때문에 block 없이 `inner()` 메서드를 실행할 수 있다.

```java
public class LockTest {
    Lock lock = new ReentrantLock();

    public void outer() {
        lock.lock(); // 첫 번째 획득
        try {
            inner(); // 내부에서도 락을 다시 획득
        } finally {
            lock.unlock(); // outer의 unlock
        }
    }

    public void inner() {
        lock.lock(); // 같은 스레드가 두 번째로 획득
        try {
            // ...
        } finally {
            lock.unlock(); // inner의 unlock
        }
    }
}
```

재진입이 안 됬다면 `outer()`에서 락을 잡고 `inner()`에서 다시 락을 잡으려고 시도하면 같은 스레드라도 락을 얻지 못하고 영원히 기다리는 데드락이 발생할 수도 있다.

ReentrantLock도 재진입을 지원한다. `outer()`가 락을 얻고 (`lock()` 호출) 그 안에서 `inner()`도 다시 락을 얻지만 (`lock()` 호출) 같은 스레드이므로 데드락 등의 문제가 발생하지 않는다.

내부적으로 ReentrantLock은 락을 잡은 스레드의 ID를 기억하고, 같은 스레드가 다시 `lock()`을 호출하면 내부 카운터만 증가시킨다. `unlock()`이 호출되면 카운터를 감소시키며 카운터가 0이 될 때 락이 해제된다.

### 공정 모드
모든 스레드가 작업 수행할 기회를 공평하게 갖는 것. 즉 스레드 간의 락 획득 순서를 보장하는 것을 말한다. 다른 스레드에게 우선 순위가 밀려 자원을 계속 할당받지 못하는 starvation을 방지할 수 있다.

ReentrantLock 생성자 파라미터를 통해 공정 모드를 설정할 수 있다.

**[생성자 보기]**
```java
public class ReentrantLock implements Lock, java.io.Serializable {
    private final Sync sync;
    ...
    public ReentrantLock() {
        sync = new NonfairSync(); // 비공정 모드
    }

    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync(); // 매개변수가 true일 경우 공정 모드
    }
    ...
}
```
- `synchronized` 와 구분 짓는 가장 큰 특징. `synchronized`는 만약 특정 스레드에 락이 필요한 순간 락 해제가 발생하면 대기열을 건너뛰는 새치기 같은 불공정한 일이 벌어질 수 있다.
- ReentrantLock은 생성자 파라미터를 통해 공정 모드를 설정할 수 있다.
    - 디폴트 생성 시 비공정 모드로 동작한다.
    - 생성자에 `true`로 넘기는 경우 가장 오래 기다린 스레드에 락을 부여하도록 공정성을 보장한다.
    
    ```java
    // 공정한 락 생성
    private final ReentrantLock fairLock = new ReentrantLock(true);
    // 비공정한 락 생성 (기본값)
    private final ReentrantLock unfairLock = new ReentrantLock(false);
    ```
    
    - 공정 모드는 starvation을 방지할 필요가 있는 경우 최선이지만, trade-off로 성능 저하 발생한다.

> 일반적으로 공정 락보다 불공정 락이 더 높은 성능을 보인다. 간혹 드물게 특정 스레드가 락 획득에 실패하여 starvation 발생할 수 있으나, 락을 획득하고자 하는 스레드를 바로 획득하는 것이 대기 중인 스레드의 순서를 고려하여 처리하는(어떤 스레드가 가장 오래 기다렸는지 확인하는 오버헤드) 시간보다 더 빠르기 때문에, 대부분 공정성의 장점보다 불공정하게 처리해서 얻는 성능상 이점이 더 크다 볼 수 있다. 
{: .prompt-tip }

### 모니터링 관련 주요 메소드

| 메소드 | 설명 |
| ------ | ------ |
| int getHoldCount() | 현재 스레드가 락을 몇 번 획득했는지(재진입 횟수) 반환 |
| boolean isHeldByCurrentThread()  | 현재 스레드의 락 보유 여부 반환 |
| boolean hasQueuedThreads() | 락 획득을 위해 대기 중인 스레드 있는지 여부 반환<br/>언제든지 대기 취소할 수 있으므고 `true` 반환한다해서<br/>조회한 스레드의 락 획득을 보장하지 않는다. |
| int getQueueLength() | 대기 중인 스레드의 개수 반환<br/>내부 데이터 구조 탐색하는 동안 스레드 수 동적으로 변경될 수 있으므로 추정값임 |
| boolean hasWaiters(Condition condition) | 지정된 컨디션에 대기 중인 스레드가 존재하는지 확인<br/>언제든 타임아웃, 인터럽트 발생할 수 있기 때문에 `true` 반환한다해서<br/> 미래의 signal이 대기 중인 스레드를 깨울 수 있음을 보장하지 않는다. |
| int getWaitQueueLength(Condition condition) | 지정된 컨디션에 대기 중인 스레드 수 추정값 반환<br/>언제든 타임아웃, 인터럽트 발생할 수 있기 때문에 실제 대기 스레드 수의 상한선 역할 |
| Collection<Thread> getWaitingThreads(Condition condition) | 지정된 컨디션에 대기 중인 스레드를 포함하는 컬렉션 반환<br/>컬렉션 구성하는 동안 실제 스레드 집합이 동적으로 변경될 수 있으므로 반환된 컬렉션은 추정값임 |
| boolean isLocked() | 락의 잠금 여부 반환 | 



### 사용 예제
SharedData 클래스는 모든 스레드가 공유할 데이터를 정의한 클래스이다.
```java
public class SharedData {
    private int value;

    public void increase() {
        value += 1;
    }
    public void print() {
        System.out.println(value);
    }
}
```
main에서 10개의 Runnable 객체를 생성해 스레드별로 공유 자원(`mySharedData`)의 `increase()`를 100번씩 호출하여 각 스레드가 100씩 증가하는 상황을 의도하는 코드를 다음과 같이 작성했다.

```java
public class TestMain {
    public static void main(String[] args) {
        final SharedData mySharedData = new SharedData();

        Runnable incrementer = () -> {
            for (int i = 0; i < 100; i++) {
                mySharedData.increase();
            }
            mySharedData.print();
        };
        
        for (int i = 0; i < 10; i++) {
            new Thread(incrementer).start();
        }
    }
}
```
```
100
200
300
571
499
400
771
871
671
971
```
`mySharedData`를 공유하는 10개 스레드가 시분할 방식으로 번갈아가며 실행하여 결과가 매번 조금씩 달라지며 의도한 결과가 보장되지 않는다.

ReentrantLock을 도입해보자.
```java
public class TestMain {
    public static void main(String[] args) {
        final SharedData mySharedData = new SharedData();
        final Lock lock = new ReentrantLock();

        Runnable incrementer = () -> {
            lock.lock();
            try {
                for (int i = 0; i < 100; i++) {
                    mySharedData.increase();
                }
                mySharedData.print();
            } finally {
                lock.unlock();
            }
        };

        for (int i = 0; i < 10; i++) {
            new Thread(incrementer).start();
        }
    }
}
```
Lock 인스턴스를 만들고 동기화가 필요한 코드의 앞 뒤에 `lock()`, `unlock()`을 호출한다.

`lock()`을 걸었으면 `unlock()`을 반드시 호출해줘야 한다. 임계 영역이 끝나더라도 `unlock()` 호출되기 전까지는 스레드의 잠금 상태가 영원히 유지되기 때문이다. 따라서 어떤 예외가 발생하더라도 반드시 `unlock()`이 호출되도록 `try-finally`문 사용을 권장한다.

```
100
200
300
400
500
600
700
800
900
1000
```

## ReentrantReadWriteLock
읽기, 쓰기 작업을 분리하여 처리하는 `ReadWriteLock` 인터페이스의 구현체이다. 읽기는 여러 스레드가 동시에 수행할 수 있지만 쓰기는 한 스레드가 독점하여 수행되어야 할 때 사용된다. (읽기에는 공유적이고 쓰기에는 베타적인 락)

특정 컬렉션을 사용할 때 동시성 개선에 사용될 수 있다. 단, 컬렉션이 크고 쓰기보다 읽기 작업이 많고 동기화 오버헤드보다 더 큰 오버헤드를 가지는 작업인 경우 유용하다.

### 동작 규칙
1. Read Lock
    - 여러 스레드가 동시에 획득 가능
    - (다른 스레드가) 쓰기 락이 획득한 상태에서 읽기 락 획득 불가
    - 쓰기 락 획득한 스레드는 읽기 락 획득 가능 (재진입성)
    
    ```java
    ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    lock.writeLock().lock(); // 쓰기 락
    ...
    lock.readLock().lock();  // 같은 스레드라면 획득 가능
    ```
    - 락의 강등 (write → read 다운그레이드)
        - 쓰기 락을 획득한 상태에서 읽기 락을 획득한 경우 쓰기 락을 해제하여 락의 등급을 강등시킬 수 있다.
        - 즉, 스레드가 데이터를 업데이트한 후 읽기 락을 유지할 수 있으므로 다른 스레드가 데이터 읽는 것을 막을 수 있다. 그러나 오랫동안 읽기 락을 유지하면 다른 스레드가 데이터 읽는 것을 막아 성능 저하를 초래할 수 있다.
    
    ```java
    lock.writeLock().lock();
    try {
        // do something...
        lock.readLock().lock();
    } finally {
        lock.writeLock().unlock();  // 락의 강등
        // do something...
        lock.readLock().unlock();
    }
    ```
    
2. Write Lock
    - 한 번에 하나의 스레드만 획득 가능 (베타적 접근 보장)
    - (다른 스레드가) 읽기 락이 획득한 상태에서 쓰기 락 획득 불가
    - 읽기 락 획득한 스레드는 쓰기 락 획득 불가 (read → write 업그레이드 불가)
        - 읽기 락은 여러 스레드가 보유할 수 있기 때문에 업그레이드 불가!
        - 쓰기 락 획득하려면 락을 잠시 풀었다가 다시 잡아야 한다. 그러나 중간에 다른 스레드가 끼어들 수 있으므로 위험한 동작으로 간주된다.
    
    ```java
    ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    lock.readLock().lock(); // 읽기 락
    ...
    // 쓰기 락 얻기 위해 모든 read lock 해제되길 기다리는데
    // 자기 자신도 read lock 잡고 있어서 절대 풀리지 않음
    lock.writeLock().lock();  // 데드락 발생 가능
    ```

## StampedLock
Java 8부터 도입된 새로운 락으로, ReadWriteLock의 성능을 개선한 버전이다.

읽기, 쓰기 접근 제어를 위해 3가지 모드를 제공하는 권한 기반의 락으로, 가장 큰 특징은 낙관적 읽기(Optimistic Reading) 모드를 제공한다는 점이다.

다른 구현체와 달리 `Lock` 또는 `ReadWriteLock` 인터페이스를 직접 구현하지 않는다.

`StampedLock`의 상태는 버전과 모드로 구성되고 락 획득 메서드는 락 상태에 대한 접근을 제어할 수 있고 락의 상태를 나타내는 식별자인 스탬프를 반환한다.

락 해제 및 변환 메서드는 스탬프를 매개변수로 받아 락 상태와 일치하지 않으면 실패한다.

### 모드
1. 쓰기 모드 : `writeLock()` 사용 시
   
    ```java
    long stamp = lock.writeLock(); // 배타적 락 획득
    try {
        // 쓰기 작업 수행
    } finally {
        lock.unlockWrite(stamp);
    }
    ```
2. 읽기 모드 : `readLock()` 사용 시

    ```java
    long stamp = lock.readLock(); // 공유 락 획득
    try {
        // 읽기 작업 수행
    } finally {
        lock.unlockRead(stamp);
    }
    ```

3. 낙관적 읽기 모드 : `tryOptimisticRead()` 사용 시

    ```java
    long stamp = lock.tryOptimisticRead(); // 락 획득 없이 낙관적으로 읽기
    // 데이터 읽기
    if (!lock.validate(stamp)) { // 도중에 데이터가 변경되었는지 확인
        // 일반 읽기 모드로 전환
        stamp = lock.readLock();
        try {
            // 데이터 다시 읽기
        } finally {
            lock.unlockRead(stamp);
        }
    }
    ```

#### 모드 업그레이드
`tryConvertToWriteLock(long stamp)` 메서드를 통해 아래 상황에서 쓰기 모드로의 업그레이드를 지원한다.

- 이미 쓰기 모드인 경우
- 읽기 모드이고 다른 읽기 스레드가 없는 경우
- 낙관적 읽기 모드이고 락이 사용한 경우

### 주의사항
- 재진입 불가하므로 락이 걸린 상태에서 락을 획득하려고 하면 안 된다.
- 스케줄링 시 읽기 모드나 쓰기 모드와 같은 특정 모드를 우선시 하지 않는다.
- `try~` 메서드는 공정성을 보장하지 않을 수 있다.


## References
- https://jhkimmm.tistory.com/36
- https://velog.io/@be_have98/Java-ReenterentLock
- https://zion830.tistory.com/57
- https://vnthf.github.io/blog/Java-java.util.concurrent.locks/
- gpt4o