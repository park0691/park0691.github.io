---
title: Java 동시성 제어 기법 (4) - Lock-Free, CAS(Compare-And-Swap), Atomic
date: 2025-09-09 12:00 +0900
categories: Java
tags: atomic, cas, nonblocking, lock
---

## 기존 Lock 방식의 한계
`Semaphore`, `Mutex`, `synchronized`와 같은 전통적인 락 기반 동기화 기법들은 한 스레드가 임계 영역(Critical Section)의 실행을 독점한다.

이러한 방식은 스레드 안전성을 보장하지만 다음과 같은 근본적인 한계를 가진다.

1. 블로킹 `blocking`과 컨텍스트 스위칭으로 인한 성능 저하
    - 한 스레드가 임계 영역을 점유하기 때문에 락을 획득하지 못한 다른 스레드는 락을 획득할 때까지 대기해야 한다. 즉, 해당 락에 접근하는 다른 스레드들은 `blocking` 상태에 들어가기 때문에 아무 작업도 하지 못한 채 자원을 낭비한다.
    - 스레드가 대기 상태로 전환되거나 대기 중이던 스레드가 다시 실행 가능 상태로 바뀔 때 컨텍스트 스위칭이 발생한다. 컨텍스트 스위칭은 스레드의 상태를 저장하고 다른 스레드의 상태를 로드하는 등 많은 CPU 연산을 소모하는 무거운 작업으로 스레드 경쟁이 심할수록 이 오버헤드는 급격히 증가하여 시스템 전체 성능을 저하시킨다.

2. 교착 상태 `deadlock` 및 우선순위 역전의 위험성
    - 둘 이상의 스레드가 서로가 점유한 락을 무한정 기다리는 데드락이 발생할 수 있다.
    - 우선순위가 낮은 스레드가 락을 점유하고 있을 때 우선순위가 더 높은 스레드가 그 락을 기다리느라 실행되지 못하는 현상으로 시스템의 반응성이 급격히 떨어질 수 있다.

따라서 간단한 작업을 처리하는 경우, 상대적으로 효율적으로 처리할 수 있는 `non-blocking`, `Lock-Free`한 방법으로 동기화 문제를 해결하기 위해 Java에서 구현한 것이 `Atomic` 클래스이며, 이 동작의 핵심 원리는 `CAS(Compare And Swap)`에 있다.


> **Lock-Free 알고리즘**<br/>
> 여러 스레드가 공유 데이터에 접근할 때, 전통적인 락을 사용하지 않고 **원자적 연산**을 통해 데이터의 일관성을 보장하는 병렬 프로그래밍 기법이다.
> 
> 한 스레드가 지연되거나 멈추더라도 전체 시스템의 진행이 멈추는 것을 방지하는 것이 핵심 목표다.
> 
> 이 알고리즘은 CAS(Compare-And-Swap)와 같은 원자적 연산을 통해 스레드 간의 충돌을 감지하고 해결하며 동시성(Concurrency)과 확장성(Scalability)을 높이는 데 사용된다.
> 
> **[시스템 전체의 안정성 증대]**<br/>
> 특정 스레드의 느려짐이나 멈춤이 다른 스레드의 작업에 영향을 미치지 않아 시스템이 멈추지 않고 안정적으로 작업을 처리할 수 있다.
> 
> **[교착 상태 원천 방지]**<br/>
> 스레드들이 서로를 기다리며 멈추는 블로킹 상태가 없으므로, 락으로 인해 발생하는 교착 상태(deadlock)가 발생하지 않는다.
> 
> **[높은 성능 및 확장성]**<br/>
> 락을 획득, 해제하는 과정에서 발생하는 컨텍스트 스위칭 비용이 없어, `synchronized` 같은 락 기반 방식보다 훨씬 빠르다.
{: .prompt-tip }
> 

Then 언제 Lock-Free 알고리즘을 사용하는 게 좋은가??

## CAS(Compare-And-Swap) 연산
CPU가 직접 실행하는 하나의 원자적인 명령어다. 말 그대로 **'값을 비교하고(Compare), 조건이 맞으면 교체(Swap)하는'** 동작이 중간에 끊어지지 않고 한 번에 처리된다.

핵심 아이디어는 **<u>내가 값을 읽은 시점과 지금 값을 쓰려는 시점 사이에 다른 스레드가 값을 변경하지 않은 경우</u>**에만 업데이트하는 것이다. 즉, **<u>현재 메모리의 값과 내가 아는 값을 비교해서 같다면 값을 업데이트한다.</u>** 아는 값과 다르다면 이미 다른 스레드가 값을 변경한 것이다. 이를 통해 여러 스레드가 동시에 값을 변경하려 할 때 발생하는 경쟁 상태(Race Condition)를 감지하고 안전하게 처리할 수 있다.

결론적으로 CAS는 락(Lock)을 사용하지 않아 락으로 인한 성능 저하나 교착 상태(Deadlock) 문제 없이, 높은 성능으로 데이터의 일관성과 동기화를 보장하는 강력한 기법이다.

### 동작 원리
실제 CAS는 하드웨어 명령어지만, 그 동작 원리를 이해하기 쉽게 `synchronized`를 이용해 Java 코드로 논리적 흐름을 시뮬레이션하면 다음과 같다.

```java
@ThreadSafe
public class SimulatedCAS {
    @GuardedBy("this") private int value;

    public synchronized int get() { return value; }
    
    // 현재 메모리 값(value)이 예상 값(expectedValue)과 같을 때만 
    // 새로운 값(newValue)으로 교체하는 CAS의 핵심 로직
    public synchronized int compareAndSwap(int expectedValue, int newValue) {
        int oldValue = value;
        if (oldValue == expectedValue)
            value = newValue;
        return oldValue;
    }
}
```
_출처 : Java Concurrecy in Practice_

CAS 연산은 **수행 이전값/예상값(expectedValue), 새로운 값(newValue)**을 요구하며, 다음과 같이 동작한다.

1. 메모리의 현재 값이 **<u>수행 이전값/예상값(expectedValue)과 동일한지 비교</u>**한다.
    - 이 과정을 통해 내가 값을 읽은 시점 이후 다른 스레드가 값을 변경했는지(즉, `Race Condition`이 발생했는지) 확인한다.
2. 비교 결과 동일하면(경쟁이 없었다면) 값을 **새로운 값(newValue)**으로 교체한다.
3. 비교 결과 동일하지 않으면 아무 작업도 하지 않고 실패한다.

### CAS 알고리즘 : 재시도 루프
실제 환경에서는 CAS 연산의 성공을 보장하기 위해, 실패 시 재시도하는 '알고리즘' 형태로 사용한다. 외부에서는 루프를 돌면서 CAS 연산이 성공할 때까지 이 과정을 반복하여, 어떤 스레드도 다른 스레드를 기다리며 멈추지 않고(Lock-Free) 동기화 처리를 완료할 수 있다.

```java
@ThreadSafe
public class CasCounter {
    private SimulatedCAS value;

    public int increment() {
        int v;
        do {
            v = value.get();
        } while(v != value.compareAndSwap(v, v + 1));
        return v + 1;
    }
}
```
_출처 : Java Concurrecy in Practice_

위 코드는 `do-while` 루프를 사용해 CAS가 성공할 때까지 업데이트를 반복한다. 핵심은 while 조건문이다.

- 성공 시 : 다른 스레드의 경합이 없었다면, `compareAndSwap(v, v + 1)`는 성공하고 이전 값인 `v`를 반환한다. `v != v`는 false가 되어 루프를 빠져나온다.
- 실패 시 : 만약 다른 스레드가 먼저 값을 변경했다면, `compareAndSwap(v, v + 1)`은 실패하고 변경된 값을 반환한다. 이 값은 `v`와 다르므로(`v != v`는 true), 루프는 처음부터 다시 시작하여 최신 값을 읽어 업데이트를 재시도한다

**[기아(Starvation) 문제]**<br/>
그러나 이 방식은 해당 스레드가 Starvation에 빠질 수 있다. 경쟁이 매우 치열한 경우(다른 스레드들이 동시에 같은 변수에 대해 CAS를 수행하면), 운이 없는 특정 스레드는 CAS 연산에 계속 실패하거나, 성공하는 데 오랜 시간이 걸릴 수 있다.

**[그럼에도 Lock-Free인 이유]**<br/>
그럼에도 불구하고 CAS 알고리즘이 Lock-Free로 불리는 이유는, 내 스레드가 실패하더라도 시스템 전체적으로는 누군가 계속 성공(다른 스레드는 성공했음을 의미)하며 앞으로 나아가기 때문이다.

즉, 한 스레드의 지연이 시스템 전체를 멈추지 않으며, 이는 하드웨어가 락 없이 진정한 원자적 연산을 보장해주기에 가능한 것이다.

### CAS In Java
JVM은 CAS 연산을 호출받았을 때 해당 하드웨어에 가장 효과적인 방법으로 처리한다. CAS 연산을 직접 지원하는 OS의 경우 CAS 연산 호출 부분을 직접 해당하는 기계어 코드로 변환해 실행한다. 하드웨어에서 지원하지 않는 최악의 경우 JVM 자체적으로 스핀 락을 사용해 CAS 연산을 구현한다.

저수준의 CAS 연산은 `AtomicInteger`와 같이 `java.util.concurrent.atomic` 패키지의 Atomic Variables 클래스를 통해 제공한다.

## Atomic
멀티 스레드 환경에서 락 없이 원자적 연산을 수행하도록 하는 Java에서 지원하는 클래스다.

`synchronized`와 다르게 논블로킹 상태에서 원자성을 보장하며, 더 적은 비용으로 스레드 안전성을 보장한다.

Java 5부터 도입된 `java.util.concurrent.atomic` 패키지를 통해 CAS 연산을 지원하는 여러 Atomic 클래스를 제공한다.

### 핵심 원리 : CAS(Compare-And-Swap)
Atomic 클래스는 내부적으로 **CAS(Compare-And-Swap)**라는 하드웨어 수준의 원자적 명령어를 사용한다. 즉, "내가 기억하는 값과 현재 메모리 값이 일치한다면, 새로운 값으로 교체해줘"라는 로직을 하나의 원자적 동작으로 처리한다.

이러한 낙관적 락(Optimistic Locking) 접근 방식을 통해, 여러 스레드가 동시에 값을 변경하려 할 때 값비싼 락을 걸고 스레드를 대기시키는 대신, 일단 시도해보고 실패하면 재시도하는 방식으로 높은 성능을 유지한다.

### 종류
1. 스칼라<br/>
단일 변수에 대한 원자적 연산을 지원한다.

    - AtomicInteger: `int` 타입 변수
    - AtomicLong: `long` 타입 변수
    - AtomicBoolean: `boolean` 타입 변수
    - AtomicReference: 임의의 객체 참조

2. 배열<br/>
배열의 각 요소에 대한 원자적 연산을 지원한다.

    - AtomicIntegerArray: `int[]` 배열
    - AtomicLongArray: `long[]` 배열
    - AtomicReferenceArray: `Object[]` 배열

3. 필드 업데이터<br/>
기존 객체의 `volatile` 필드에 대해 원자적으로 업데이트할 수 있도록 돕는다.

    - AtomicIntegerFieldUpdater
    - AtomicLongFieldUpdater
    - AtomicReferenceFieldUpdater

4. 복합 변수<br/>
   ABA 문제 해결이나 값의 누적을 위해 사용된다.

    - AtomicStampedReference: 값과 버전(스탬프)을 함께 관리하여 ABA 문제를 해결한다.
    - LongAdder, DoubleAdder: 여러 스레드가 동시에 값을 더하는 시나리오에 특화되어 있으며, 높은 경합 상황에서 AtomicLong/AtomicDouble보다 뛰어난 성능을 보인다.


### 주요 메소드

| 메소드 | 설명 |
| ------- | ------- |
| T get()  | 현재 값을 반환한다. |
| void set(T newValue)  | `newValue`로 값을 업데이트한다. |
| T getAndSet(T newValue)   | 원자적으로 값을 업데이트하고 원래의 값을 반환한다. |
| boolean compareAndSet(T expectedValue, T newValue)  | Atomic 클래스의 핵심이 되는 저수준 `CAS` 연산.<br/>현재 값이 예상하는 값(`expectedValue`)과 동일하다면 값을 업데이트한 후<br/> `true`를 반환한다. 같지 않다면 업데이트하지 않고 `false`를 반환한다. |


- Number 타입의 경우 값의 연산을 할 수 있도록 메서드를 추가로 제공한다.

| 메소드 | 설명 |
| ------- | ------- |
| T incrementAndGet() | 값에 1을 더한 후, 그 결과를 반환한다.(`++i`) |
| T decrementAndGet() | 값에서 1을 뺀 후, 그 결과를 반환한다.(`--i`) |
| T addAndGet(T delta) | 값에 `delta` 만큼 더한 후, 그 결과를 반환한다.(`i += delta`) |
| T getAndIncrement() | 값에 1을 더하고, 더하기 전의 기존 값을 반환한다.(`i++`) |
| T getAndDecrement() | 값에서 1을 빼고, 빼기 전의 기존 값을 반환한다.(`i--`) |
| T getAndAdd(T delta) | 값에 `delta` 만큼 더하고, 더하기 전의 기존 값을 반환한다. |
| T getAndSet(T newValue) | 값을 `newValue`로 설정하고, 설정하기 전의 기존 값을 반환한다. |

### 사용 예제 : AtomicInteger
AtomicInteger 클래스를 사용하는 다음 코드를 보자.

```java
public class AtomicIntegerTest {

    private static int count;

    public static void main(String[] args) throws InterruptedException {
        AtomicInteger atomicCount = new AtomicInteger(0);
        int threadCount = 2;
        Thread[] threads = new Thread[threadCount];

        for (int t = 0; t < threadCount; t++) {
            threads[t] = new Thread(() -> {
                for (int i = 0; i < 100000; i++) {
                    count++;
                    atomicCount.incrementAndGet();
                }
            });
            threads[t].start();
        }

        Thread.sleep(5000);
        System.out.println("atomic 결과 : " + atomicCount.get());
        System.out.println("int 결과 : " + count);
    }
}
```

**[실행 결과]**

![images](https://1drv.ms/i/c/9251ef56e0951664/IQS2HCK-76uiQbie1JjDBewXAYORzcKM0rlB0MvEPtEljC8)

`atomicCount`는 의도대로 200000이 출력되지만, `count`는 동기화되지 않아 의도하지 않은 값이 출력되는 것을 확인할 수 있다.

**[AtomicInteger 뜯어보기]**
```java
public class AtomicInteger extends Number implements java.io.Serializable {
    ...
    private static final jdk.internal.misc.Unsafe U = jdk.internal.misc.Unsafe.getUnsafe();
    private static final long VALUE = U.objectFieldOffset(AtomicInteger.class, "value");
    
    private volatile int value;
    ...
    public final int get() {
        return value;
    }
    
    public final void set(int newValue) {
        value = newValue;
    }
    ...
    public final boolean compareAndSet(int expectedValue, int newValue) {
        return U.compareAndSetInt(this, VALUE, expectedValue, newValue);
    }
    ...
    public final int incrementAndGet() {
    	return U.getAndAddInt(this, VALUE, 1) + 1;
    }
}
```

1. `value` 변수에 `volatile` 키워드가 붙어 있다.
    - CAS 연산 외의 메서드를 위한 가시성 보장
        - `AtomicInteger` 의 모든 메서드가 CAS 루프를 사용하는 것은 아니다. `get()`, `set(int newValue)` 는  CAS 연산이 적용되지 않기 때문에 메인 메모리에서 값을 읽고 쓰지 않고 CPU 캐시에서 읽고 쓰기 때문에 가시성 문제가 발생할 수 있다.
        - 즉, `volatile`은 가장 단순한 읽기/쓰기 연산에서도 가시성을 완벽하게 지키기 위한 필수 장치다.
    - `happens-before` 보장
        - `volatile` 로 선언함으로써 컴파일러나 CPU가 코드 순서를 임의로 바꾸는 것을 방지한다.
        - `AtomicInteger`의 핵심인 `compareAndSet()` 연산도 내부적으로 `volatile`과 동일한 효과를 내도록 구현되어 있으나, `value` 필드 자체에 `volatile`을 명시하는 것은 이 클래스의 모든 연산이 Java 메모리 모델의 엄격한 메모리 동기화 규칙을 따른다는 것을 명확히 하는 일종의 계약이며, 코드의 안정성을 더욱 견고하게 만들어 준다.

2. JVM에서 제공하는 `Unsafe` 클래스를 사용한다.
    - 이 클래스는 `native` 메서드로 이루어져 Java에서 C/C++ 처럼 메모리를 직접 제어할 수 있다.
    - 왜 사용했을까?<br/>
    `Atomic` 클래스는 락보다 훨씬 빠른 CPU의 CAS 연산을 활용하고자 했다. 그러나 일반 Java는 이 하드웨어 명령어를 호출할 방법이 없었으므로, JDK 개발자들은 숨겨진 `Unsafe` 클래스를 통해 CPU의 원자적 명령어를 직접 호출하는 기능을 구현했다.

    > `Unsafe`의 대안 : **VarHandle**<br>
    > `Unsafe`는 비표준이고 위험하다는 문제 때문에 Java 9부터 이를 대체하는 `VarHandle` 클래스가 도입되었다.
    {: .prompt-warning }
    >

**[Unsafe 뜯어보기]**
```java
public final class Unsafe {
    ...
    @HotSpotIntrinsicCandidate
    public final int getAndAddInt(Object o, long offset, int delta) {
        int v;
        do {
            v = getIntVolatile(o, offset);
        } while (!weakCompareAndSetInt(o, offset, v, v + delta));
        return v;
    }
}
```
위 코드는 `AtomicInteger`의 `incrementAndGet()` 메서드 내부에서 호출되는 `Unsafe`클래스의 메서드이다.

1. 최신 값 읽기 (7L)<br/> `v = getIntVolatile(o, offset);`
    - `getIntVolatile()`을 사용해 메인 메모리에서 **가장 최신 값**을 읽어와 `v`에 저장한다.
        - `volatile` 읽기를 통해 다른 스레드가 값을 변경했더라도 그 최신 값을 볼 수 있도록 **가시성**을 보장하는 것을 확인할 수 있다.
    - `o`는 AtomicInteger 객체 자신을, `offset`은 그 객체 안에서 `value` 필드가 위치한 메모리 상의 주소(거리)를 의미한다.

2. 원자적 업데이트 시도 (8L)<br/> `while (!weakCompareAndSetInt(o, offset, v, v + delta));`
    - `weakCompareAndSetInt`라는 CAS 연산을 시도합니다. 이 연산은 CPU에게 다음과 같은 요청을 하나의 원자적 동작으로 보낸다.
        - *"메모리(o, offset)의 현재 값이 내가 좀 전에 읽은 값(v)과 같으면, 그 값을 새로운 값(v + delta)으로 바꿔줘."*
    - **성공** : 그 사이에 다른 스레드의 변경이 없었다면, 값은 성공적으로 업데이트되고 `true`가 반환되어 `while` 루프가 종료된다.
    - **실패** : 만약 다른 스레드가 먼저 값을 변경했다면, 현재 메모리 값은 `v`와 다르므로 업데이트는 실패하고 `false`가 반환된다. 루프는 처음부터 다시 시작하여 변경된 최신 값을 읽어온다.


## References
- https://en.wikipedia.org/wiki/Compare-and-swap
- https://ozt88.tistory.com/38
- https://highright96.tistory.com/106
- https://www.baeldung.com/lock-free-programming
- https://velog.io/@joosing/high-performance-multithreaded-sync-techniques-compare-and-swap
- https://mangkyu.tistory.com/415
- https://www.baeldung.com/lock-free-programming
- https://jaehyeon48.github.io/computer-architecture/cache-memory/
- https://dalichoi.tistory.com/entry/Lock-Free-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-%EC%82%B4%ED%8E%B4%EB%B3%B4%EA%B8%B0CAS-Volatile-Java-Atomic-Variables
- https://docs.oracle.com/javase/8/docs/api/index.html?java/util/concurrent/atomic/AtomicBoolean.html
- https://docs.oracle.com/javase/tutorial/essential/concurrency/atomic.html
- gpt4o
- Gemini 2.5 Pro