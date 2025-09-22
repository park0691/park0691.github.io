---
title: Java 동시성 제어 기법 (1) - 모니터를 이용한 Synchronized
date: 2024-12-27 12:00 +0900
categories: Java
tags: synchronized, monitor, synchronization, 동기화
---

## Monitor
![monitor](https://1drv.ms/i/c/9251ef56e0951664/IQSJaFRLs14xQbufQNyA8gCPAXDXMV05voNN9CpH0JHvvvI)

### 왜 쓰나
세마포어, 뮤텍스 잘못 쓰면 타이밍 에러(Timing Error) 같은 문제 자주 발생한다.

> e.g. 이진 세마포어를 사용하여 1로 초기화한 경우 `wait()`을 수행한 뒤 `signal()`을 수행해야 하는 일련의 순서를 지켜야 한다. 호출 순서가 잘못 되면 데드락, 리소스 누수 등의 문제가 발생한다.

이러한 문제를 해결하기 위해 동기화 도구를 하이 레벨의 언어로 좀 더 편리하게 사용할 수 있도록 모니터 기법을 제공한다.

### 무엇인가
- 상호 배제와 조건의 동기화를 포함하는 고수준의 동기화 구조 (Mutex와 Condition Variable를 캡슐화)
    - 동시성 프로그래밍에서 스레드들이 상호 배제와 특정 `condition = false` 되면 wait 하는 동기화 구조
    - 모니터에서 사용하는 상호 배제 방식은 뮤텍스. 모니터는 <u>mutex(lock) 오브젝트와 condition variable로 구성</u>된다. condition variable은 <u>특정 상태를 기다리는 스레드들의 컨테이너</u><br/>
    
    > **Condition Variable**<br/>
    > 특정 컨디션이 참이 될때까지 기다리게 하는 역할. 모니터 안에서 코드가 특정 상태가 참이 될때까지 기다릴 때 그 코드는 condition variable 위에서 기다린다.
    
    - 여러 상태 중 하나가 `true` 되면 waiter를 깨우는 시그널을(signal) 보내거나, 모든 waiter를 깨우는 시그널을(broadcast) 보낸다.
    - 스레드들이 (exclusive access를 다시 획득하고 작업을 재개하기 전) 특정 상태 충족을 기다리기 위해 임시적으로 exclusive access를 포기하는 메커니즘도 제공
    - 스레드 세이프한 클래스, 오브젝트 혹은 모듈. 하나 이상의 스레드에 의패 variable, method에 안전하게 접근을 허가하기 위해 mutex를 감싼 개념
    
- 뮤텍스, 세마포어보다 좀 더 고 수준(high level, 추상화된)의 동시성 제어 기법. 프로그래밍 언어로 세마포어를 좀 더 편리하게 사용할 수 있도록 인터페이스를 제공하여 동시성 제어에 대한 개발자의 부담을 덜어준다.

- 시스템 콜과 유사한 개념. 시스템 콜은 사용자로부터 커널을 보호하기 위한 인터페이스.  커피 머신을 사용자가 직접 만지면 고장날 가능성이 높은 것처럼, OS 자원을 사용자가 마음대로 사용하게 두면 시스템 자원을 손상시킬 수 있다. 따라서 시스템 자원을 사용자로부터 숨기고 사용자의 요구사항을 처리할 수 있는 인터페이스만 제공하는데 이를 시스템 콜이라 한다. 모니터도 <u>보호할 자원을 Critical Section으로 만들고 Critical Section에서 수행할 수 있는 인터페이스만 제공</u>하여 자원을 보호한다.

- 모니터 내부에 공유 데이터와 공유 데이터에 접근하는 코드를 모두 정의한다. 그래서 공유 데이터에 접근하려면 모니터 안에 정의된 코드를 통해서만 접근할 수 있으며, 정의된 코드가 공유 데이터에 동시에 접근하면 모니터가 애초에 막는다. 그래서 개발자는 락을 걸고 푸는 코드를 추가할 필요가 없다. 모니터가 알아서 제어해 준다.

### 구성 요소
#### 모니터 타입, Monitor Type
- 프로그래머가 정의한 연산의 집합인 추상 자료형(Abstract Data Type)으로 상호 배제를 제공한다.
```
monitor A {
	/* shared variable declarations */
	function P1(...) {
		...
	}
	...
	function Pn(...) {
		...
	}
	initialization_code(...) {
		...
	}
}
```
- 모니터 타입은 다른 프로세스가 직접 사용할 수 없다.
	- 오직 모니터 내에 정의된 코드를 통해서만 공유 데이터(모니터 내의 지역 변수, 타입 매개 변수 등)에 접근 가능하다.
- 모니터 구조물은 항상 하나의 프로세스(스레드)만 활성화되도록 보장한다.
	- 동기화 제약 조건을 개발자가 직접 코딩하지 않아도 된다.

#### 조건 변수, Conditional Variables
##### 상호 배제의 한계
![monitor](https://1drv.ms/i/c/9251ef56e0951664/IQQud1q7B-fjQJq3IazFy9GyAbNsawmoF--yI72JotSvm0c?width=1024)

다음과 같이 락을 거는 상호 배제만으로 동기화에 한계가 있다. 락은 지금 자원을 쓰고 있는지만 판단할 뿐, 연산을 수행하기 전 <u>어떤 조건(e.g. 버퍼가 비어 있지 않은가?)이 만족할 때까지 기다려야 하는 논리는 구현할 수 없다</u>. (condition이 참인지 체크)

**[Spin-based Approach : CPU 시간 낭비]**<br/>
이를 위해 전역 변수를 하나 선언하고 작업 스레드가 완료될 때 이 값을 바꾼다. waiter는 해당 값이 바뀔 때까지 체크하는 `while` 루프를 고려할 수 있다. 그러나 작업 스레드가 완료될 때까지 waiter는 계속 CPU를 점유하여 자원 효율이 좋지 않다.

##### Producer / Consumer (Bound Buffer) 문제
- Producer
    - 데이터를 생산한다.
    - 버퍼에 데이터를 넣는다.
- Consumer
    - 버퍼에서 데이터를 꺼내서 소비한다.
- Bounded Buffer는 공유 자원으로 동기화가 필요하다.
    - 생산 속도가 소비 속도보다 빠른 경우가 많아 생산된 데이터는 바로 소비될 수 없다. 이를 보완하기 위해 유한한 크기의 버퍼에 데이터를 보관한다.
    - Producer는 버퍼가 가득 차면 더 이상 넣을 수 없고, Consumer는 버퍼가 비면 빼낼 수 없다.
    - Producer, Consumer가 동시에 접근할 수 있으므로 atomic하게 접근되어야 한다.

```java
global RingBuffer queue; // A thread-unsafe ring-buffer of tasks.

// Method representing each producer thread's behavior:
public method producer() {
    while (true) {
        task myTask = ...; // 프로듀서는 새로운 작업을 생성한다.
        while (queue.isFull()) { } // 큐가 가득 안 찰 때까지 Busy-wait
        queue.enqueue(myTask); // 큐에 새 작업을 추가한다.
    }
}

// Method representing each consumer thread's behavior:
public method consumer() {
    while (true) {
        while (queue.isEmpty()) { } // 큐가 찰 때 까지 Busy-wait
        myTask = queue.dequeue(); // 큐에서 작업을 꺼낸다.
        doStuff(myTask); // 꺼낸 작업을 수행한다.
    }
}
```

- 동시 접근 가능한 코드
    - `queue.isEmpty(), queue.isFull(), queue.enqueue(), queue.dequeue()`

**[문제 상황]**
- Producer / Consumer 스레드가 `enqueue()`, `dequeue()` 동시에 수행한다면 Race Condition 야기한다.
    - Critical Section에 여러 스레드가 동시에 진입하여 큐의 멤버 변수(size, beginning, ending position, assignment, allocation of queue elements)에 대한 동시 업데이트가 발생
    - 멤버 변수 변경 도중 Context Switch에 의해 데이터의 일관성 깨짐
- 한 Consumer 스레드가 다른 Consumer 스레드가 `Busy-wait`을 나가고 `dequeue()` 호출하는 사이에 큐를 empty 하게 만들고, 빈 큐에 `dequeue()` 시도하면 에러를 야기한다.
- 한 Producer 스레드가 다른 Producer 스레드가 `Busy-wait`을 나가고 `enqueue()` 호출하는 사이에 큐를 가득 차게 만들고, 다른 Producer가 가득 찬 큐에 `enqueue()` 시도하면 에러를 야기한다.

##### Spin-Wait
`Busy-wait` 체크 구간 사이에 Mutex 락을 추가하여 락을 획득했다 푸는 방법으로 임계 영역을 보호하도록 한다.

```java
global RingBuffer queue; // A thread-unsafe ring-buffer of tasks.
global Lock queueLock; // A Mutex for the ring-buffer of tasks.

// Method representing each producer thread's behavior:
public method producer() {
    while (true) {
        task myTask = ...; // 프로듀서는 새로운 작업을 생성한다.
        
        queueLock.acquire(); // Busy-wait 체크를 위한 Lock 획득
        while (queue.isFull()) {
            queueLock.release(); // 락이 필요한 Consumer 스레드에게 기회를 주기 위해 락을 일시적으로 푼다.
            queueLock.acquire(); // 다음 queue.isFull() 호출을 위해 락을 재획득
        } // 큐가 가득 안 찰 때까지 Busy-wait
        queue.enqueue(myTask); // 큐에 새 작업을 추가한다.
        queueLock.release()
    }
}

// Method representing each consumer thread's behavior:
public method consumer() {
    while (true) {
        queueLock.acquire(); // Busy-wait 체크를 위한 Lock 획득
        while (queue.isEmpty()) {
            queueLock.release(); // 락이 필요한(작업을 추가하는) Producer 스레드에게 기회를 주기 위해 락을 일시적으로 푼다.
            queueLock.acquire(); // 다음 queue.isEmpty() 호출을 위해 락을 재획득
        } // 큐가 찰 때 까지 Busy-wait
        myTask = queue.dequeue(); // 큐에서 작업을 꺼낸다.
        doStuff(myTask); // 나와서 작업을 수행한다.
    }
}
```

**[CPU 자원 낭비]**<br/>
이 방법은 inconsistent 상태가 발생하지 않도록 보장한다. 그러나 불필요한 busy-waiting 으로 CPU 자원을 낭비한다.

만약 큐가 비어있고 Producer 스레드가 아무 것도 큐에 넣지 않더라도 Consumer 스레드는 불필요한 busy-waiting을 해야 한다. 마찬가지로 Consumer 스레드가 오랫동안 block 되고 큐가 가득 차 있더라도 Producer 스레드는 항상 busy-waiting 해야 한다.

Producer는 큐가 Non-full 될 때까지 block 되게 만들고, Consumer는 큐가 Non-empty 될 때까지 block 되게 만들면 이 문제를 해결할 수 있다.

Mutex 자신 또한 Spin Lock이 될 수 있다. Spin Lock은 락을 획득하기 위해 Busy-waiting을 수반하지만 CPU 자원 낭비 문제를 해결할 수 있는 개념이다. 코드에서 사용한 `queueLock`은 Spin Lock이 아니라고 가정한다.

##### Definition
- 모니터 타입만으로 동기화를 구현하는 데 충분하지 않으므로 `condition` 변수를 통해 동기화 매커니즘을 제공한다.
    - 스레드를 잠들게 하거나, 자고 있는 스레드를 깨우는 기능을 지원 (기존 Mutex/Semaphore는 lock을 잡은 채로 대기했으나 `wait()`가 내부적으로 락 해제 후 블록시킴)

```
condition x, y;
x.wait();
x.signal();
```

- 조건이 충족될때까지 <u>기다리는 여러 스레드를 저장하는 큐(Waiting Queue)</u>가 있다.
    - Waiting Queue : 조건이 충족되길 기다리는 스레드들이 대기 상태로 머무르는 큐

> 어떻게 Condition을 기다리는가?
> - Waiting on the condition
>    - 원하는 상태가 아닐 때, 스레드 자신을 명시적인 큐에 넣는다.
> - Signaling on the condition
>    - 다른 스레드가 상태를 변경하면 대기 중인 스레드 중 하나를 깨우고 계속 진행할 수 있게 한다.
>
{: .prompt-tip }

- 또한 큐에 접근하는 3가지 기능을 제공한다.
    - `wait()` : release lock, go to sleep, reacquire lock
    - `signal()` : wake up a waiter, if any
    - `broadcast()` : wake up all waiters
    
- `x.wait()`를 호출한 프로세스는 다른 프로세스가 `x.signal()`을 호출할 때까지 대기
- `x.signal()`는 대기 중인 프로세스를 재개시킨다.
  - 다른 프로세스에서 `x.signal()` 호출하기 전까지 대기 상태
- `memoryless`
    - 조건 변수는 이전에 signal 온 사실을 저장하지 않으므로, `wait()`은 이전에 signal 이 왔던 안왔던 안왔던 무조건 스레드를 잠들게 한다.

##### 주요 연산
스레드가 Condition Variable에서 기다리는 동안 그 스레드는 모니터를 소유할 수 없다. 다른 스레드는 모니터에 들어가 모니터의 상태를 바꿀 수 있다. 대부분 다른 스레드들은 assertion이 참이 되었다고 Condition Variable에게 시그널을 보낸다.

- `wait(Condition_Variable cv, Mutex m)`
    - `blocks the calling thread` (호출한 스레드를 블럭시키며 **<u>락을 포기</u>**한다.)
        - 잠들기 전 락을 해제해야 다른 스레드가 락을 획득할 수 있으므로 락을 해제하고 잠든다.
        - `wait()` 호출될 때 Mutex 락은 획득된 상태이어야 한다.
    - 스레드에 의해 호출되는데 assertion이 참이 될 때까지 기다리게 한다. 스레드가 기다리는 동안 모니터를 소유할 수 없다.
    - Atomic : 이 연산은 `atomic`해야 한다. 즉 다음 순서가 진행될 동안 외부에 의해 방해 받으면 안 된다. 그렇지 않으면 release lock과 sleep 사이에 다른 스레드가 끼어들어 문제 일으킬 수 있다.
        1. `Mutex m`락을 해제한다.
        2. 호출한 현재 스레드를 Running 상태에서 Condition Variable의 Wait Queue로 보내어 그 스레드를 재운다. (Sleep / Block) 
        3. 컨텍스트는 다른 스레드에게 양도된다. (CPU 자원 양도)
    - 그 스레드가 signal 받아서 다시 일을 한다면 자동으로 `Mutex m` 락을 획득한다.
        - 다시 락을 얻어야만 Critical Section에 진입할 수 있으므로!
    
- `signal(Condition_Variable cv)` / `notify(Condition_Variable cv)`
    - assertion이 참이 되면 스레드가 호출하는 함수
    - Condition Variable에서 대기 중인 스레드가 있는 경우 한 스레드를 깨운다.
      - 하나 이상의 스레드가 Condition Variable의 Sleep Queue 에서 Ready Queue로 이동하게 한다.
    - 보통 `Mutex m`을 놓아주기 전에 signal 하는 게 좋은 구현이지만, signal 전에 `Mutex m`을 놓아주기도 한다.
    
- `broadcast(Condition_Variable cv)` / `notifyAll(Condition_Variable cv)`
    - `signal` 함수와 동일한 조건이지만 Wait Queue의 모든 스레드를 깨운다. (Wait Queue를 비움)
    - Condition Variable와 연관된 스레드가 2개 이상이면 signal 대신 broadcast를 호출한다.

**[1:N 관계]**<br/>
여러 개의 Condition Variable은 하나의 Mutex에 연관될 수 있지만 그 역은 성립하지 않는다. (Mutex와 Condition Variable의 관계는 1:N) 똑같은 Mutex를 필요로 하는 같은 Variable 내에 다른 Condition을 기다리고 싶어하는 다른 스레드가 존재하기 때문이다.

Producer / Consumer 예시에서 큐는 Unique한 Mutex에 의해 보호되지만 <u>다른 Condition Variable을 사용</u>하는 것이 좋다. (각각의 모니터 사용) Producer 스레드는 `Mutex m`, `Condition Variable c_full`(큐가 가득 안 찰 때까지 블록하는 조건 변수)을 사용해서 기다린다면 Consumer 스레드는 같은 Mutex를 사용하지만 다른 Condition Variable인 `c_empty`(큐가 빌 때까지 블록하는 조건 변수)를 사용하는 다른 모니터에서 기다리는 게 좋다. (⇒ **<u>Consumer는 Producer만 깨우고, Producer는 Consumer만 깨우게 할 수 있기 때문이다.</u>**)

##### Signal Semantics
만약 프로세스 P의 `signal()` 이 대기 중인(sleep) 프로세스 Q를 깨웠을 때(resume) 프로세스 P는 계속 실행되어야 할까? or 프로세스 Q가 실행되어야 할까?

- Hoare monitors (original), signal and wait
    - waiter에게 Lock을 넘겨주고 signaller는 sleep 한다. (signal 주고 대기함으로써 다른 스레드가 먼저 수행하도록 양보)
        - `signal()`하면 caller에서 대기 중인 스레드로 즉시 전환되고, 조건 변수에 의해 깨워진 스레드가 즉시 실행을 이어받는다.
        - P는 Q가 모니터를 빠져나가거나 다른 condition을 위해 기다리기 전까지 wait
    - waiter가 우선권 가짐
    - waiter가 resume 될 때 waiter가 기대하던 condition 이 보장된다. (`while`문 반드시 사용할 필요는 없다. `if` 문으로 충분)

    ```
    1. 스레드 P가 signal() 호출
    2. 조건을 기다리던 스레드 Q가 깨어남. 스레드 P는 대기 상태로 전환 (wait)
    3. 스레드 Q가 실행
    4. 스레드 Q가 종료되면, 스레드 P가 다시 실행
    ```
- Mesa monitors, signal and continue
    - waiter를 ready queue에 넣고 signaller가 계속 Lock을 점유한다. (signal 주고 내가 먼저 실행한 뒤 sleep 하여 다른 스레드가 수행되도록 함.)
        - `signal()`하면 대기 중인 스레드를 준비 큐에 둔다. 그러나 signaller는 모니터 안에서 계속된다.
        - Q는 P가 모니터를 빠져나가거나 다른 condition을 위해 기다리기 전까지 wait
    - waiter는 신호 받고 나서 스케줄러에 의해 나중에 실행된다.
    - waiter가 바로 깨어나지 않기 때문에 대기하는 동안 다른 스레드에 의해 condition이 깨질 가능성이 존재한다. (resume 될 때 조건이 반드시 참이 아니다. `while`문 condition 체크 반드시 필요)

    ```
    1. 스레드 P가 signal() 호출
    2. 조건을 기다리던 스레드 Q가 Ready 상태로 변경. 스레드 P는 계속 실행 중
    3. 스레드 P는 락을 푼 뒤, 스레드 Q가 스케줄되어 실행
    ```

### 사용법
```java
acquire(m); // 모니터 락 획득
while (!p) { // Condition, Predicate, Assertion (p)가 false 될 때까지
    wait(cv, m);
}

signal(cv2);
// 또는
notifyAll(cv2); // cv2는 cv와 같거나 다를 수 있다.

// 모니터 락 해제
release(m);
```

#### Producer / Consumer (Bound Buffer) 문제 해결
```java
global RingBuffer queue; // A thread-unsafe ring-buffer of tasks.
global Lock queueLock; // ring-buffer를 위한 조건 변수
global CV queueEmptyCV; // Consumer 스레드 위한 조건 변수
global CV queueFullCV; // Producer 스레드 위한 조건 변수

// Method representing each producer thread's behavior:
public method producer() {
    while (true) {
        task myTask = ...; // 프로듀서는 새로운 작업을 생성한다.
        
        queueLock.acquire(); // Predicate 체크를 위한 Lock 획득
        while (queue.isFull()) { // Predicate (큐가 가득 차야 한다)
            // 스레딩 시스템이 queueLock을 원자적으로 해제하고
            // 이 스레드를 queueFullCV에 넣고 재운다. (sleep)
            wait(queueFullCV, queueLock);
            // wait()은 Predicate 체크를 위해 queueLock을 자동으로 재획득한다.
        } // 큐가 가득 안 찰 때까지 Busy-wait
        
        // Critical Section (큐가 가득 차지 않기를 요구하는), queueLock을 획득한 상태
        queue.enqueue(myTask); // 큐에 새 작업을 추가한다.
        
        // 큐가 비지 않으므로 signal 또는 broadcast
        // (큐가 비지 않길 기다리는 하나 이상의 Consumer 스레드를 깨운다.)
        signal(queueEmptyCV); // - 또는 - notifyAll(queueEmptyCV);
        
        // End of Critical Section
        queueLock.release(); // 락을 해제한다.
    }
}

// Method representing each consumer thread's behavior:
public method consumer() {
    while (true) {
        queueLock.acquire(); // Predicate 체크를 위한 Lock 획득
        while (queue.isEmpty()) { // Predicate (큐가 비어야 한다)
            // 스레딩 시스템이 queueLock을 원자적으로 해제하고
            // 이 스레드를 queueEmptyCV에 넣고 재운다. (sleep)
            wait(queueEmptyCV, queueLock);
            // wait()은 Predicate 체크를 위해 queueLock을 자동으로 재획득한다.
        } // 큐가 찰 때 까지 Busy-wait
        
        // Critical Section (큐가 가득 비지 않길 요구하는), queueLock을 획득한 상태
        myTask = queue.dequeue(); // 큐에서 작업을 꺼낸다.
        
        // 큐가 가득 차지 않으므로 signal 또는 broadcast
        // (큐가 가득차지 않기를 기다리는 하나 이상의 Producer 스레드를 깨운다.)
        signal(queueFullCV); // - 또는 - notifyAll(queueFullCV);
        
        // End of Critical Section
        queueLock.release(); // 락을 해제한다.
        
        doStuff(myTask); // 나와서 작업을 수행한다.
    }
}
```

### 한계
- 스레드 스캐줄링 제어 불가
    - 어떤 스레드를 깨울지 직접 선택 불가능. 대기 중인 스레드의 우선 순위를 고려하지 않음
- Starvation 가능성
    - 특정 스레드가 계속 스케줄되지 않아 대기만 하게 되는 상황 발생 가능

## Monitor In Java
- Java에서 동기화와 관련된 동시성 제어의 기본 단위
- 모든 Java 객체는 모니터를 가진다.
- 여러 스레드가 임계 영역에 진입할 때 JVM은 모니터를 사용해 스레드 간 동기화를 제공한다.
- 상호 배제, 협력 두 가지 동기화 기능을 제공하며, 이를 위해 뮤텍스와 조건 변수를 제공한다.

> 왜 모니터라 부를까?<br/>
> 스레드가 자원에 어떻게 접근하는지 모니터링하기 때문이다.
{: .prompt-tip }

> 장점
> - `synchronized` 키워드로 간편하게 구현할 수 있다. 세마포어를 직접 구현할 필요 없으며 `synchronized` 키워드로 임계 영역을 표시함으로써 모니터 영역을 만들 수 있다.
> - 메소드(코드) 블록을 나갈 때 자동으로 모니터 해제
> - 조건 변수를 사용하여 스레드간 협력을 구현할 수 있다.
{: .prompt-warning }

> 단점
> - 잘못 사용하면 교착 상태에 빠질 수 있다.
> - `synchronized` 키워드는 성능 오버헤드를 발생시킬 수 있다.
{: .prompt-danger }

### 상호 배제(Mutual Exclusion)
Java에서 모니터 매커니즘의 구현은 `entry-set`, `wait-set` 두 가지 개념에 의존한다.

모니터는 건물의 전용 공간(Exclusive Room)으로 비유할 수 있다. 말 그대로 한 번에 한 사람만이 전용 공간에 있을 수 있다.

- 모니터는 두 개의 방과 복도를 가진 건물
- 동기화되는 자원은 전용 공간 `Exclusive Room`( → `Critical Section`)
- `wait-set`은 대기실, `entry-set`은 복도
- 스레드는 전용 공간에 들어가고 싶은 사람들

전용 공간 `Critical Section`에 들어가고자 하는 스레드는 먼저 스케줄러를 기다리는 복도 `entry-set`로 간다. 따라서 스케줄러는 스레드를 선택하여 전용 공간으로 보낸다.

JVM 스케줄러는 우선순위 기반 스케줄링 알고리즘을 사용한다. 두 스레드가 동일한 우선 순위를 갖는 경우 JVM은 FIFO 방식을 사용한다.

> 단계별로 정리하자면
> - 건물에 들어가기 : 모니터에 들어가기
> - 전용 공간 입장 : 모니터 획득
> - 전용 공간에 있는 것 : 모니터를 소유
> - 전용 공간 나가기 : 모니터 해제
> - 건물에서 나가기 : 모니터에서 나가기
{: .prompt-warning }


> 상호 배제, Mutual Exclusion<br/>
> 객체가 가진 모니터 락을 퉁해 한 번에 한 스레드만 공유 자원에 접근하도록 허용하여 데이터의 일관성, 안전성을 보장한다.
{: .prompt-tip }

### 모니터의 대기 구조
![monitor](https://1drv.ms/i/c/9251ef56e0951664/IQTC78oZZxPhTJHCmHRZmbocAXptY_YkXnhnLa7bfKczVXc)

![monitor](https://1drv.ms/i/c/9251ef56e0951664/IQREguO9djAtSK0FE5vM3eOHAToC4B6MM5bzTySUrnlQYw8)

#### EntrySet
- 모니터 Lock을 획득하기 위해 대기 중인 스레드를 모아 놓은 자료 구조
- 스레드가 Lock을 점유한 경우, 경쟁에서 밀려난 다른 스레드는 EntrySet에 들어간다.
- EntrySet에 있는 스레드들은 Lock이 반납될 때까지 기다리며 락이 반납되면 EntrySet 중 한 스레드가 락을 획득하고 임계 영역으로 진입한다.

#### WaitSet
- `wait()` 호출한 스레드들이 대기하는 자료 구조
- Lock을 획득해서 작업하는 도중 조건이 만족하지 않아 대기해야 하는 경우 WaitSet에 들어가며, 특정 조건이 만족할 때까지 WaitSet에서 대기한다.
- Lock을 사용하던 스레드가 WaitSet에 들어가 대기할 때 Lock은 해제된다.
- 활성 스레드가 `notify(), notifyAll()`을 호출하면 <u>EntrySet으로 이동</u>해서 다시 Lock을 획득할 수 있다. 또한 `notify()` 호출할 때 EntrySet에도 `notify()`를 보낸다. 이 때 개체를 담는 컬렉션은 Set이므로 별도의 우선순위는 없다.

### 협력(Cooperation) - wait()과 notify(), Condition Variable

`synchronized` 는 상호 배제를 보장하지만 `busy-waiting` 없이 특정 조건이 만족될 때까지 기다리게 하거나, 다른 스레드에게 '내가 멈출테니 다른 스레드 실행해라' 와 같은 스레드 간 협력이 불가하다.

`synchronized` 블록에서 사용되는 `java.lang.Object` 클래스의 `wait(), notify(), notifyAll()` 메소드와 모니터의 조건 변수(Condition Variable)를 통해 스레드 간 협력을 가능하게 한다. A 스레드가 특정 조건을 만족하지 못해 대기하면 B 스레드가 특정 조건을 만족시킨 뒤 A 스레드를 깨우는 방식으로 협력한다.

- `wait(), notify(), notifyAll()`은 모두 `synchronized` 블록 안에서만 사용할 수 있다. (즉, 모니터 락을 확보한 상태에서만 동작한다.)
- Java 모니터는 <u>오직 한 개의 조건 변수</u>만 가질 수 있다.
- 조건 변수는 `wait(), notify(), notifyAll()` 과 함께 작동하며 특정 조건이 충족될 때까지 다른 스레드를 대기시킬 수 있다.
- 스레드가 특정 조건에 충족되지 않으면 `wait()` 메서드를 호출해 조건 변수의 대기 셋(wait-set)에서 대기한다. (대기 상태로 전환)
- 스레드가 특정 조건을 만족해서 `notify()` 또는 `notifyAll()` 메서드를 호출해 해당 조건 변수의 대기 셋으로부터 스레드를 깨운다.
  - 어떤 스레드가 선택될지는 알 수 없다.

- 조건 변수를 통해 스레드 간 `wait()`과 `notify()`를 조절하면서 경쟁 조건(Race Condition) 문제를 방지할 수 있다.


#### Signalling
스레드가 `notify()` 호출했을 때, 어떤 스레드가 모니터를 먼저 소유하는가에 따라 두 가지로 구분할 수 있다. **<u>Java에서는 `Signal and Continue` 방식을 취한다.</u>**

##### Signal and Wait
- 현재 모니터를 소유한 스레드가 `wait()`하면 자신을 일시 중단하고 Lock을 해제한 후 WaitSet에 들어간다.
- 깨우는 스레드가 `notify()` 또는 `notifyAll()`을 호출하면 WaitSet에 있는 대기 중인 스레드를 깨우고, 깨운 스레드(caller)는 Lock을 해제하고 대기한다.
- 대기에서 깨어난 스레드가 Lock을 획득한 후 모든 작업을 마치고 Lock을 해제하면 깨운 스레드가 Lock을 획득한 후 작업을 계속 진행한다.
- 대기 스레드와 깨운 스레드 사이에 다른 스레드가 모니터를 소유할 수 없도록 원자적 실행이 보장되어야 한다.

##### Signal and Continue
- 현재 모니터를 소유한 스레드가 `wait()`하면 자신을 일시 중단하고 Lock을 해제한 후 WaitSet에 들어간다.
- 깨우는 스레드가 `notify()` 또는 `notifyAll()`을 호출하면 WaitSet에 있는 대기 중인 스레드를 깨운다. 이 때 일어난 스레드들은 <u>EntrySet으로 이동</u>한다.
- 깨우는 스레드는 Lock을 계속 유지하면서 모든 작업을 완료하고 Lock을 해제하면 <u>EntrySet에 대기하고 있는 모든 스레드가 Lock을 획득하기 위해 경쟁</u>한다.

### Synchronized
- 명시적인이지 않은 Java에 내장된 락으로 암묵적인 락 `Intrinsic Lock`, 모니터 락 `Monitor Lock` 이라고도 한다.
- 임계 영역을 설정할 때 사용하는 Java 키워드
- `synchronized` 블록은 해당 객체의 모니터를 획득할 수 있으며, 모니터를 획득한 단 하나의 스레드만 임계 영역에 접근 가능하고 그 외 다른 스레드들은 블로킹 `blocking` 되어 대기 상태가 된다.
- `synchronized` 블록을 빠져나오면 모니터 락이 해제되고, 대기 중인 다른 스레드 중 하나가 모니터 락을 얻고 임계 영역에 진입하여 작업을 수행하는 방식으로 상호 배제가 보장된다.
- `synchronized`가 적용된 하나의 메서드만 호출해도 같은 모니터의 모든 synchronized 메서드까지 락에 잠기게 되어 락이 해제될 때까지 접근 불가하다.
- `sleep()`을 실행한 스레드는 동기화 영역에서 대기 중이더라도 획득한 락을 놓거나 해제하지 않는다.
- `synchronized`의 동기화 영역에 진입하지 못하고 대기 중인 스레드는 인터럽트되지 않는다.

- 메서드나 코드 블록에 적용한다.
	- synchronized method
		- instance
		- static
    - synchronized block
    	- instance
    	- class
```java
synchronized(obj) {
    // Critical Section - 코드 블록
}
```
```java
synchronized void methodA() {
    // Critical Section - 메서드
}
```

#### 동기화 방법
**[인스턴스 메서드 동기화]**
- 인스턴스 단위로 모니터가 동작하며 <u>동일한 인스턴스 안에서 `synchronized`가 적용된 곳은 하나의 락을 공유</u>한다.
  - 인스턴스가 여러 개일 경우 인스턴스별로 모니터 객체를 가진다.
  - 인스턴스 메서드는 `this`가 모니터


***(인스턴스가 하나일 때)***

```java
public class Method {

    public static void main(String[] args) {
        Method sync = new Method();
        Thread thread1 = new Thread(() -> {
            System.out.println("스레드1 시작 " + LocalDateTime.now());
            sync.syncMethod1("스레드1");
            System.out.println("스레드1 종료 " + LocalDateTime.now());
        });

        Thread thread2 = new Thread(() -> {
            System.out.println("스레드2 시작 " + LocalDateTime.now());
            sync.syncMethod2("스레드2");
            System.out.println("스레드2 종료 " + LocalDateTime.now());
        });

        thread1.start();
        thread2.start();
    }

    private synchronized void syncMethod1(String msg) {
        System.out.println(msg + "의 syncMethod1 실행중" + LocalDateTime.now());
        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    private synchronized void syncMethod2(String msg) {
        System.out.println(msg + "의 syncMethod2 실행중" + LocalDateTime.now());
        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

```
스레드2 시작 2025-05-21T09:40:14.045238900
스레드1 시작 2025-05-21T09:40:14.045238900
스레드2의 syncMethod2 실행중2025-05-21T09:40:14.051235200
스레드2 종료 2025-05-21T09:40:19.053719100
스레드1의 syncMethod1 실행중2025-05-21T09:40:19.053719100
스레드1 종료 2025-05-21T09:40:24.057917200
```

스레드2의 `synchMethod2()` 가 종료된 다음 스레드1이 `synchMethod2()` 호출한 것을 확인할 수 있다.

***(인스턴스가 각각일 때)***

```java
public class Method {
    public static void main(String[] args) {
        Method method = new Method();
        Method method2 = new Method();
        Thread thread1 = new Thread(() -> {
            System.out.println("스레드1 시작 " + LocalDateTime.now());
            method.syncMethod1("스레드1");
            System.out.println("스레드1 종료 " + LocalDateTime.now());
        });

        Thread thread2 = new Thread(() -> {
            System.out.println("스레드2 시작 " + LocalDateTime.now());
            method2.syncMethod2("스레드2");
            System.out.println("스레드2 종료 " + LocalDateTime.now());
        });

        thread1.start();
        thread2.start();
    }
    ...

```

```
스레드1 시작 2025-05-21T09:45:41.935953500
스레드2 시작 2025-05-21T09:45:41.935953500
스레드2의 syncMethod2 실행중2025-05-21T09:45:41.945393300
스레드1의 syncMethod1 실행중2025-05-21T09:45:41.945393300
스레드1 종료 2025-05-21T09:45:46.961895700
스레드2 종료 2025-05-21T09:45:46.961895700
```

이 경우 모니터를 공유하지 않기 때문에 스레드 간 동기화가 발생하지 않는다.


**[정적 메서드 동기화]**
- <u>클래스 단위로 모니터가 동작하며 `synchronized`가 적용된 곳은 하나의 락을 공유</u>한다.
- 인스턴스와는 별개의 모니터를 가지고 임계 영역을 동기화하기 때문에 인스턴스 단위로 메서드를 호출할지라도 클래스 단위로 스레드간 공유한다.
  - 정적 메서드는 클래스가 모니터

- 클래스는 메모리에 오직 하나만 존재하므로 하나의 모니터를 공유해서 동기화하고자 할 때 사용할 수 있다.

```java
public class MyClass {
    public static synchronized void syncMethod1() {
        // Critical Section
    }
    public static synchronized void syncMethod2() {
        // Critical Section
    }
    
    public static void main(String[] args) {
        // 클래스 단위로 스레드 간 공유
        MyClass.syncMethod1();
        MyClass.syncMethod1();
    }
}
```

**[인스턴스 블록 동기화]**

- 인스턴스 단위로 모니터가 동작하며 `synchronized`가 적용된 곳은 하나의 락을 공유한다.
- 모든 인스턴스가 모니터를 가지기 때문에 여러 인스턴스로 구분해서 동기화를 구성할 수 있다.
- 클래스 인스턴스가 여러 개일 경우 인스턴스 별로 모니터 객체를 가지며 스레드는 모니터 별로 락을 획득해서 `synchronized` 영역을 진입하고 빠져나올 때 락을 해제할 수 있다.

```java
public class MyClass {
    public void syncMethod1() {
        synchronized(this){
            // Critical Section
        }
    }
    public void syncMethod2() {
        synchronized(this){
            // Critical Section
        }
    }

    public static void main(String[] args) {
    	// 다른 인스턴스이므로 다른 모니터
        new MyClass().syncMethod1();
        new MyClass().syncMethod1();
    }
}
```

***synchronized(this)***

```java
public class Block1 {
    public static void main(String[] args) {
        Block1 block = new Block1();

        Thread thread1 = new Thread(() -> {
            System.out.println("스레드1 시작 " + LocalDateTime.now());
            block.syncBlockMethod1("스레드1");
            System.out.println("스레드1 종료 " + LocalDateTime.now());
        });

        Thread thread2 = new Thread(() -> {
            System.out.println("스레드2 시작 " + LocalDateTime.now());
            block.syncBlockMethod2("스레드2");
            System.out.println("스레드2 종료 " + LocalDateTime.now());
        });
        thread1.start();
        thread2.start();
    }

    private void syncBlockMethod1(String msg) {
        synchronized (this) {
            System.out.println(msg + "의 syncBlockMethod1 실행중" + LocalDateTime.now());
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    private void syncBlockMethod2(String msg) {
        synchronized (this) {
            System.out.println(msg + "의 syncBlockMethod2 실행중" + LocalDateTime.now());
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

```
스레드1 시작 2025-05-21T09:55:43.038674100
스레드2 시작 2025-05-21T09:55:43.038674100
스레드1의 syncBlockMethod1 실행중2025-05-21T09:55:43.051406100
스레드1 종료 2025-05-21T09:55:48.053132100
스레드2의 syncBlockMethod2 실행중2025-05-21T09:55:48.053132100
스레드2 종료 2025-05-21T09:55:53.058890600
```

synchronized 파라미터로 `this`를 사용하면 모든 synchronized 블럭에 락이 걸린다. 여러 스레드가 들어와서 다른 synchronized 블럭을 호출해도 기다려야 한다.

***synchronized(Object)***

synchronized의 파라미터로 Object를 사용하면 블록마다 다른 락이 걸리게 되어 효율적으로 코드를 작성할 수 있다.

```java
public class Block2 {
    private final Object o1 = new Object();
    private final Object o2 = new Object();

    public static void main(String[] args) {
        Block2 block = new Block2();

        Thread thread1 = new Thread(() -> {
            System.out.println("스레드1 시작 " + LocalDateTime.now());
            block.syncBlockMethod1("스레드1");
            System.out.println("스레드1 종료 " + LocalDateTime.now());
        });

        Thread thread2 = new Thread(() -> {
            System.out.println("스레드2 시작 " + LocalDateTime.now());
            block.syncBlockMethod2("스레드2");
            System.out.println("스레드2 종료 " + LocalDateTime.now());
        });
        thread1.start();
        thread2.start();
    }

    private void syncBlockMethod1(String msg) {
        synchronized (o1) {
            System.out.println(msg + "의 syncBlockMethod1 실행중" + LocalDateTime.now());
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    private void syncBlockMethod2(String msg) {
        synchronized (o2) {
            System.out.println(msg + "의 syncBlockMethod2 실행중" + LocalDateTime.now());
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

```
스레드2 시작 2025-05-21T09:58:35.796714200
스레드1 시작 2025-05-21T09:58:35.797713
스레드2의 syncBlockMethod2 실행중2025-05-21T09:58:35.811845200
스레드1의 syncBlockMethod1 실행중2025-05-21T09:58:35.811845200
스레드2 종료 2025-05-21T09:58:40.825170700
스레드1 종료 2025-05-21T09:58:40.825170700
```

스레드1과 2의 동기화가 되지 않은 것을 확인할 수 있다.

**[정적 블록 동기화]**

- 클래스 단위로 모니터가 동작하며 `synchronized`가 적용된 곳은 하나의 락을 공유한다.
- 모든 클래스가 모니터를 가지기 때문에 모니터를 여러 클래스로 구분해서 동기화를 구성할 수 있다.
- 클래스 모니터가 여러 개일 경우 스레드는 모니터 별로 락을 획득해서 synchronized 영역을 진입하고 빠져 나올 때 락을 해제 할 수 있다.

```java
public class MyClass {
    public static void syncMethod1() {
        synchronized(MyClass.class){
            // Critical Section
        }
    }
    public static void syncMethod2() {
        synchronized(YourClass.class){
            // Critical Section
        }
    }

    public static void main(String[] args) {
    	// 클래스 인스턴스는 메모리에 1개. 동일한 모니터
        MyClass.syncMethod1();
        MyClass.syncMethod1();
    }
}
```

#### 특성
**[재진입성]**

- 모니터 내에서 `synchronized` 영역에 들어간 스레드가 다시 같은 모니터 영역으로 들어갈 수 있는데, 이를 '모니터 재진입' 이라 한다.


```java
class Parent {
    public synchronized void method1() {
        // Critical Section
    }
}
class Child extends Parent {
    public synchronized void method2() {
        super.method1();
    }
}

Child c = new Child();
c.method2();
```

- 재진입 가능하다는 것은 락의 획득이 호출 단위가 아닌 스레드 단위로 일어난다는 것을 의미하며 이미 락을 획득한 스레드는 같은 락을 얻기 위해 <u>대기할 필요 없이 `synchronized` 블록을 만났을 때 같은 락을 확보하고 진입</u>한다.

**[상속]**
- 상속 관계에서 자식은 부모의 락과 동일한 락을 가진다.
- 동기화된 메서드에서 다른 동기화된 메서드를 호출하는 경우 이미 락을 가지고 있는 스레드와 같은 락을 확보하기 때문에  재진입 시 데드락이 발생하지 않고 정상적으로 진행할 수 있다.

```java
public class MyClass {
    static class Parent {
        public synchronized void method() {
            System.out.println("Parent method");
        }
    }
    static class Child extends Parent {
        @Override
        public synchronized void method() {
            System.out.println("start super");
            super.method();
            System.out.println("finish super");
        }
    }

    public static void main(String[] args) {
        Child child = new Child();
        child.method();
    }
}
```

**[가시성]**
- 한 스레드가 공유 자원에 수정, 쓰기 작업할 때 다른 스레드가 수정한 내용이 보이는 '가시성' 을 `synchronized`는 지원한다.
  - CPU 캐시에 수정한 변수 값을 쓰지 않고 메모리에 직접 쓰는 것을 의미
  - 멀티 스레드 환경에서 작업 시 CPU가 참조하는 메모리가 달라 데이터 불일치 방지
- **락 획득 시** : `synchronized` 블록에 진입할 때, 자신의 CPU 캐시를 무효화하고 메인 메모리에서 직접 값을 다시 읽는다.
- **락 해제 시** : `synchronized` 블록을 빠져나갈 때, 자신의 CPU 캐시에 있던 모든 변경 사항을 메인 메모리로 `flush` 한다.

```java
public class Volatile {
    private static boolean stopRequested;

    public static void main(String[] args) throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
            Integer i = 0;
            while (!stopRequested) {
                synchronized(i) {
                    i++;
                }
            }
        });
        backgroundThread.start();

        Thread.sleep(1000);
        stopRequested = true;
    }
}
```

위 코드에서 synchronized 블럭이 없으면 무한 루프를 돌 수 있다. synchronized 키워드는 블럭에 진입 전 CPU 캐시 메모리와 메인 메모리를 즉시 동기화 해준다.

**[비공정성]**

- `synchronized` 영역에 진입하지 못한 다른 스레드가 모니터를 획득하는 데 순서가 정해져 있지 않다.
- 기아 상태에 빠진 스레드가 나올 수 있지만 OS 레벨에서 적절히 처리한다.

#### 동작 과정
##### 상호 배제
![monitor](https://1drv.ms/i/c/9251ef56e0951664/IQSRr5X7FzzRRJ6JSNHI6nVqAZfufLHVUBJRqa7XuA4VAxU?width=1024)
_출처 : https://velog.io/@appti/%EC%9E%90%EB%B0%94-synchronized_
(1) 스레드 T1이 `synchronized` 영역에 접근해 EntrySet에서 대기<br/>
(2) Mutex 락 획득 시도 → 최초 실행이므로 바로 락을 획득<br/>
(3) Critical Section에 접근하여 작업 수행

![monitor](https://1drv.ms/i/c/9251ef56e0951664/IQR7Sc7Q0CTRTpUDlinz7LcqASmlJBJHPfegzBQp6MLRwh4?width=1024)
_출처 : https://velog.io/@appti/%EC%9E%90%EB%B0%94-synchronized_
(4) 스레드 T2가 `synchronized` 영역에 접근해 EntrySet에서 대기<br/>
(5) Mutex 락 획득 시도<br/>
(6) 이미 T1이 Critical Section에 있으므로 Block

![monitor](https://1drv.ms/i/c/9251ef56e0951664/IQSQfFWYqhFJSKFP3K04msTbAZjWGRlmeXeSB9O-pVat5Hw?width=1024)
_출처 : https://velog.io/@appti/%EC%9E%90%EB%B0%94-synchronized_
(7) T1이 Critical Section에서 나와서 모니터 해제 (`wait()` 호출 또는 작업 완료하여)<br/>
(8) EntrySet에서 대기하던 T2가 Mutex 락 획득 시도<br/>
(9) (락을 획득한 스레드가 없으므로) T2는 Critical Section에 접근하여 작업을 수행

![monitor](https://1drv.ms/i/c/9251ef56e0951664/IQQYbUekhnh9R6VaSH5ncuifAZA_n4vkLDTPV84wFnXywfY?width=1024)
_출처 : https://velog.io/@appti/%EC%9E%90%EB%B0%94-synchronized_
이와 같이 상호 배제 기반으로 T1은 작업을 모두 마쳤고, T2는 새롭게 Critical Section에 도달했다.

##### 상호 협력
![monitor](https://1drv.ms/i/c/9251ef56e0951664/IQTAmRk097DlRIwZH_egbqs3AaKlWNfq_E9zaIRVAJe9new?width=1024)
_출처 : https://velog.io/@appti/%EC%9E%90%EB%B0%94-synchronized_
(1) T2가 조건(Condition Variable)을 만족하지 못해 `wait()` 호출<br/>
(2) `wait()` 호출한 T2는 WaitSet에서 대기

![monitor](https://1drv.ms/i/c/9251ef56e0951664/IQTJZodcjqU6TqqkyAehCUSnASVYVKmSfye2Cml0f78dIOM?width=1024)
_출처 : https://velog.io/@appti/%EC%9E%90%EB%B0%94-synchronized_
WaitSet에서 대기하는 스레드는 다른 스레드가 `notify()` 또는 `notifyAll()` 호출하기 전까지 깨어나지 않는다. (상호 협력, 상호 배제는 별도 동작)

다음 상황을 가정하자.
- T2는 WaitSet에서 대기
- T3는 Mutex 락을 획득하지 못해 EntrySet에서 대기 중
- T4는 Mutex 락을 획득해 Critical Section에서 작업 수행 중

![monitor](https://1drv.ms/i/c/9251ef56e0951664/IQQRm4UfUeoJSYdMt6TZ7wdZAeGnGVFmsjfV5OuhDRwacwM?width=1024)
_출처 : https://velog.io/@appti/%EC%9E%90%EB%B0%94-synchronized_
(3) T4가 조건(Condition Variable)을 만족시키고 WaitSet에 대기하는 스레드를 깨우기 위해 `notify()` 또는 `notifyAll()` 호출

- Java Monitor는 `signal and continue` 방식이므로 `notify()` 또는 `notifyAll()`을 호출하더라도 모니터를 해제하지 않고 작업을 수행함

![monitor](https://1drv.ms/i/c/9251ef56e0951664/IQSbElwklJffSIexowExHajdARoXHelx5dFR0On5NHbRpC0?width=1024)
_출처 : https://velog.io/@appti/%EC%9E%90%EB%B0%94-synchronized_
(4) Wait Set에 대기하던 T2가 EntrySet으로 이동

![monitor](https://1drv.ms/i/c/9251ef56e0951664/IQRihqOAlkezRYyfyqmQNhllAYKPJOS50XZfzH1oZoXHVQc?width=1024)
_출처 : https://velog.io/@appti/%EC%9E%90%EB%B0%94-synchronized_
(5) Critical Section에서 모든 작업을 수행한 T4는 모니터 해제<br/>
(6) EntrySet에서 대기하던 T2, T3는 Mutex 락을 획득하기 위해 경쟁<br/>
(7) 락을 획득한 스레드가 없으므로 T2, T3 중 하나의 스레드가 락을 획득해 Critical Section에 진입

![monitor](https://1drv.ms/i/c/9251ef56e0951664/IQRzi8pLijERR6SSNC2zGz3TAeH8u0zMAjOGr98Ds2sd2ag?width=1024)
_출처 : https://velog.io/@appti/%EC%9E%90%EB%B0%94-synchronized_
`synchronized`는 공정성이 없기 때문에 T2가 T3보다 오래 기다렸지만 T3가 먼저 실행될 수 있으므로 T3가 Critical Section에 먼저 진입할 수 있다. (EntrySet, WaitSet에서 다음 스레드를 선택하는 기준은 OS 스케줄러에 의해 결정됨)

#### 한계
- 선별적으로 signal() 불가하다.
    - Java 객체의 모니터는 Condition Variable을 하나만 사용하기 때문
- 블로킹 메커니즘에 의해 동시성을 제어하므로 락을 얻기 위해 많은 스레드가 동시에 락을 얻으려고 하는 경우, 시스템 전체의 성능이 저하될 수 있다.
    - 컨텍스트 스위칭 비용이 발생하기 때문


## References
- https://www.baeldung.com/cs/monitor
- https://devdebin.tistory.com/16
- https://devdebin.tistory.com/335
- https://velog.io/@bbamjoong/xuhnwflw
- https://velog.io/@appti/%EC%9E%90%EB%B0%94-synchronized
- https://howudong.tistory.com/339
- https://mini98.tistory.com/71
- https://en.wikipedia.org/wiki/Monitor_(synchronization)
- https://copycode.tistory.com/67
- https://rntlqvnf.github.io/lecture%20notes/os-4th-1/
- https://velog.io/@seanlion/monitorsync
- https://cseweb.ucsd.edu/classes/fa05/cse120/lectures/120-l6.pdf
- https://steady-coding.tistory.com/556
- Silberschatz et al. 『Operating System Concepts』. WILEY, 2020.
- gpt4o