---
title: Java 동시성 제어 기법 (3) - 메모리 가시성 문제, 그리고 volatile
date: 2025-09-03 12:00 +0900
categories: Java
tags: visibility, volatile, cache, thread
---

## Memory Visibility, 가시성
멀티 스레드 환경에서 스레드는 각자의 캐시를 가지는 다른 프로세서에 할당되어 병렬 실행될 수 있다. 스레들이 하나의 변수를 공유하면 각 프로세서의 캐시는 변수의 복사본을 소유하게 된다. 이 때, 여러 스레드에 의해 캐시에 있는 공유 변수에 읽고 쓰는 연산이 수행되면 캐시가 불일치되는 문제(동일한 변수임에도 각 스레드는 다른 값을 바라보는 현상)가 발생할 수 있다.

따라서 멀티 스레드 환경에서 여러 스레드가 동시에 변수에 접근, 수정하면 모든 스레드에게 변수의 값이 일관되게 보여지도록 가시성이 확보되어야 한다. 즉, <u>모든 스레드는 공유 자원에 대해 모두 같은 상태를 바라볼 수 있어야 한다.</u>

### 캐시 메모리 요약
![images](https://1drv.ms/i/c/9251ef56e0951664/IQSYF2zCtNVaS7VLslMJCnn3AYRAzE3A0a4ym-78DoMvKAc)
_출처 : Computer Organization and Architecture 10th Edition_

프로세서와 메모리 사이의 속도 차이를 보완하기 위해 사용되는 CPU 내부의 작고 빠른 임시 저장 공간이다. 멀티 스레드 환경, 멀티 코어 환경에서 각 CPU는 메모리에 접근하기 전, 캐시 메모리에 원하는 데이터가 있는지 먼저 확인한다. 필요한 데이터가 캐시 메모리에 있으면 메모리까지 갈 필요 없이 바로 가져다 쓸 수 있어 처리 속도가 크게 향상된다.

#### 종류 (계층 구조)
![images](https://1drv.ms/i/c/9251ef56e0951664/IQQLgWwHA4l6RakpPzl7S9ZLAV91jMwXi0Nmx0yZ6ugwyzA)
_출처 : https://www.researchgate.net/figure/A-classical-three-level-cache-hierarchy_fig1_362707415_

명령어를 처리하는 CPU와 인접할수록 데이터를 빠르게 조회할 수 있다. CPU 내의 데이터 처리를 위한 임시 저장 공간인 레지스터(Register)부터 시작하여, L1/L2/L3 캐시 순으로 데이터 조회를 시도한다.

- L1 캐시 : 용량은 가장 작지만(수십 KB) 가장 속도가 빠른 캐시로 각 CPU 코어에 존재함. 명령어 캐시(Instruction Cache)와 데이터 캐시(Data Cache)로 나누어진다.
- L2 캐시 : L1 캐시보다 크고 약간 느리며, 각 CPU 코어에 존재함. 일반적으로 64KB ~ 4MB 정도의 용량을 가짐.
- L3 캐시 : 가장 크고 느린 캐시로 CPU의 모든 코어가 공유함.

#### 한계
캐시 메모리릍 통해 데이터 접근 속도를 높여 성능을 향상시킬 수 있지만, 멀티 스레드 환경에서 한 스레드가 갱신한 값을 다른 스레드가 즉시 인지하지 못하고 캐시에 저장된 데이터를 기반으로 연산을 수행하는 가시성 문제가 발생할 수 있다.

## Memory Visibility Problem In Java
가시성을 보장하지 못하는 다음 코드를 보자

```java
public class VisibilityProblem {

    public static void main(String[] args) {
        // 스레드 객체 생성 후 시작
        SampleThread sample = new SampleThread();
        sample.start();

        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Sleep Ended!");

        // sample 객체에 있는 instanceVariable 값을 -1로 변경하고 끝낸다.
        sample.setValue(-1);
        System.out.println("Set value completed.");
    }
}

// 샘플 스레드
class SampleThread extends Thread {

    private int value = 0;

    void setValue(int value) {
        this.value = value;
    }

    @Override
    public void run() {
        // variable 값이 0이면 계속 돈다.
        while (value == 0);

        // variable 값이 변경되면, 변경된 값을 출력하고 해당 스레드는 작업이 종료된다.
        System.out.println(value);
    }
}
```
메인 스레드는 1초 후 샘플 스레드 객체의 값을 -1로 바꾸기 때문에 while 문을 빠져나와 프로그램이 종료될 것으로 예상된다.

프로그램의 수행 결과를 보면

![images](https://1drv.ms/i/c/9251ef56e0951664/IQRgr7YJbAoYSbVUj5K103zuAW5788HGSGIAZ_yx36sIGGk)

그러나 샘플 스레드는 while 문을 바로 빠져나오지 못하며, 샘플 스레드가 종료되지 못해 프로그램이 끝나지 않는다.

### 문제 원인 분석
결론부터 말하자면, 각 스레드에서 수행되는 변수의 값을 읽고 쓸 때 메인 메모리에서 참조하지 않고, CPU 캐시 메모리에서 참조하므로 main() 스레드가 수정한 값이 sample 스레드의 캐시 메모리에 바로 반영되지 않기 때문이다.

CPU 1에서 수행된 스레드를 main() 스레드, CPU 2에서 수행된 스레드를 sample 스레드라 하자.

![images](https://1drv.ms/i/c/9251ef56e0951664/IQTKE_hBR_zVQIV_DQYS3XkxAUlDkV9Bq1yj1Lb6vOXDgEE)

1. main() 스레드의 캐시 변경
    - main() 스레드는 자신에게 할당된 CPU 캐시의 value 값을 -1로 변경한다.
2. 메인 메모리로 반영 (`flush`) 
    - main() 스레드의 캐시에 있던 value 값(-1)이 메인 메모리에 기록(`flush`)된다. 이 시점은 JMM(Java Memory Model)에 의해 보장되지 않으므로 그 즉시가 아닐 수 있다. 하지만 중요한 점은 어쨌든 메인 메모리는 결국 `-1`로 변경된다는 것이다.
3. sample 스레드는 변경을 모른다.
    - sample 스레드는 이미 value의 초기값(`0`)을 자신의 CPU 캐시에 저장했으므로 메인 메모리의 value 값이 변경되었는지 굳이 확인해야 할 의무가 없다. 따라서 성능을 위해 자신의 캐시에 있는 `0` 값만 계속해서 읽게 되어 계속해서 while 문을 돌게 된다.
    - 즉, main() 스레드가 수정한 값을 sample 스레드가 언제 보게 될지 알 수 없기 때문에 가시성 문제가 발생한다.

## volatile
Java에서는 이러한 가시성 문제를 해결하기 위해 `volatile` 키워드를 사용한다. 변수를 `volatile` 로 선언하면 **<u>값을 읽거나 쓸 때 캐시 메모리를 사용하지 않고 메인 메모리에 직접 접근</u>**하게 한다. 즉, **<u>동일 시점에 모든 스레드가 동일한 값을 가지도록 동기화</u>**한다고 볼 수 있다.

**[특징]**
- 프로세서 내의 레지스터/캐시에서 지역적으로 읽기/쓰기 작업이 처리되지 않으며, 항상 메인 메모리에서 읽기/쓰기 작업이 수행된다.
- 상호 배제를 제공하지 않고도 데이터 변경의 가시성을 보장한다.
- 원자적 연산에서만 동기화를 보장한다.

### volatile 키워드로 가시성 문제 해결
여러 스레드가 사용하는 공유 변수 `value`에 `volatile` 키워드를 선언한다.

```java
class SampleThread extends Thread {

    private volatile int value = 0;

    void setValue(int value) {
    ...
```

`volatile` 키워드로 선언된 변수는 CPU 캐시 메모리를 거치지 않고 메인 메모리로 직접 읽고 쓰는 작업을 수행하게 되므로 위 코드의 수행 결과는

![images](https://1drv.ms/i/c/9251ef56e0951664/IQSzIVTkCd1eQ53CzTOHyUV5AWe5qkKMDpnfrsaINU5_gG8)

main() 스레드에 의해 변경된 메인 메모리의 값을 sample 스레드도 바라보게 되어 1초 후 변경된 숫자가 출력되며 프로그램이 정상 종료되는 것을 확인할 수 있다.

### Full Visibility of volatile
Java에서 `volatile` 키워드는 `volatile` 이 붙은 변수만을 메인 메모리에 기록할 뿐만 아니라 해당 `volatile` 변수와 함께 보여지는 모든 변수가 메인 메모리에 저장(`flush`)된다.

즉, volatile 키워드가 붙은 변수와 함께 보여지는 모든 변수의 가시성이 보장된다.

간단하게 요약하자면,

- 스레드 A가 `volatile` 변수에 쓰는 경우, `volatile` 변수 외에도 스레드 A가 볼 수 있는 모든 변수들은 메인 메모리에 함께 기록되고, 스레드 B는 그 최신값을 볼 수 있다.
- 스레드 A가 `volatile` 변수를 읽는 경우, 스레드 A에 표시되는 모든 변수는 메인 메모리에서 다시 읽힌다.

예시 코드를 보자.

```java
public class YMD {
    private int years;
    private int months;
    private volatile int days;

    public void update(int years, int months, int days){
        this.years  = years;
        this.months = months;
        this.days   = days;
    }
    
    public int totalDays() {
        int total = this.days;
        total += months * 30;
        total += years * 365;
        return total;
    }
}
```

`update()` 호출 시 `days`에 값을 쓰면 해당 스레드에서 볼 수 있는 모든 변수들 또한 메인 메모리에 기록된다. 즉, `volatile` 키워드가 붙지 않은 `years, months` 변수도 가시성이 보장된다.

- `Full Visibility`를 만족시키기 위해 메인 메모리에 쓰는 경우 `volatile` 변수를 마지막에 써야(write) 한다.

`totalDays()` 호출 시 `days` 변수를 읽어오면서 `years, months` 값도 메인 메모리에서 읽어온다.

- `Full Visibility`를 만족시키기 위해 메인 메모리에 읽는 경우 `volatile` 변수를 맨 처음에 읽어야(read) 한다.

### Instruction Reordering Challenges
JVM에서는 성능을 향상시키기 위해 실행 동작을 바꾸지 않는 선에서 프로그램의 명령어의 순서를 변경 `reordering` 할 수 있다.

```java
/** [Before] **/
int a = 1;
int b = 2;

a++;
b++;

/** [After Reordering] **/
int a = 1;
a++;

int b = 2;
b++;
```

그러나 `volatile` 변수가 포함되면 명령어를 `reordering` 하는 과정에서 문제가 발생할 수 있다.

```java
/** [Before] **/
public class YMD {
    private int years;
    private int months;
    private volatile int days;

    public void update(int years, int months, int days) {
        this.years  = years;
        this.months = months;
        this.days   = days;                // Full Visibility를 만족시키기 위해 volatile 변수는 마지막에 Write
    }

/** [After Reordering] **/
    ...
    public void update(int years, int months, int days) {
        this.days   = days;
        this.months = months;
        this.years  = years;
    }
```

명령어 `reordering` 후에는 `months, years` 변수가 값을 저장하기 전에 `days`가 먼저 값을 저장하므로 `months, years` 변수는 메인 메모리에 저장되지 않는다. (즉, `Full Visibility`를 만족할 수 없다.)

`volatile`을 사용하면 `reordering` 과정에서 문제가 발생할 수 있기 때문에 이를 방지하기 위해 Java의 `volatile`은 `Happens-Before`를 보장한다. (즉, 컴파일러의 최적화 동작으로 인해 발생할 수 있는 프로그램의 오동작을 방지한다.) <Java 5부터 도입>

> **Happens-Before**<br/>
> `volatile` 변수에 대한 읽기/쓰기 명령은 JVM에 의해 `reordering` 되지 않음을 보장한다는 의미다. 즉, `volatile` 이 일종의 경계선이 되어 JVM이 최적화를 위해 명령어의 순서를 바꾸더라도 이 경계선을 절대 넘지 않도록 보장한다.
>
> - `volatile` 쓰기 이전의 연산들은 절대 그 쓰기 이후로 재배치되지 않는다. 
    - `volatile` 변수에 대한 쓰기 이전의 명령들은 `reordering` 이후에도 `volatile` 변수에 대한 쓰기 명령 이전에 실행되도록 유지한다.
> - `volatile` 읽기 이후의 연산들은 절대 그 읽기 이전으로 재배치되지 않는다.
>    - `volatile` 변수에 대한 읽기 이후의 명령들은 재정렬 이후에도 `volatile` 변수에 대한 읽기 이후에 실행되도록 유지한다.
> - `volatile` 변수에 대한 명령 이전/이후에 존재한다는 그 전제는 반드시 지켜진다는 것이다.
>
{: .prompt-tip }

### long과 double의 원자화
JVM은 데이터를 4바이트(32비트) 단위로 처리하기 때문에 int와 int보다 작은 타입들은 한 번에 읽거나 쓸 수 있다. 즉, 하나의 명령어로 읽기, 쓰기 작업이 가능한 작업의 최소 단위이므로 작업의 중간에 다른 스레드가 끼어들 수 없다.

그러나 크기가 8바이트인 long, double 타입은 하나의 명령어로 값을 읽거나 쓸 수 없기 때문에 변수의 값을 읽는 과정에서 다른 스레드가 끼어들 여지가 있다.

다른 스레드가 끼어들지 못하게 하기 위해 변수를 읽고 쓰는 문장을 `synchronized` 블럭으로 감쌀 수도 있지만, 변수 선언 시 `volatile`을 사용함으로써 해당 변수에 대한 읽기, 쓰기 작업을 원자화할 수도 있다.

### 한계
**[원자성 부재]**<br/>
`volatile`은 변수 값을 메인 메모리에서 직접 읽고 쓰게 하여 여러 스레드가 캐시 없이 항상 최신 값을 보도록 보장할 뿐, 여러 단계로 이루어진 연산 전체를 한 번에 실행되도록 보장하지는 않는다.

멀티 스레드 환경에서 `volatile` 변수에 셋팅된 새로운 값이 이 변수가 가지던 이전 값에 의존하지 않는다면, 여러 스레드가 `volatile` 변수를 수정하면서도 메인 메모리에 존재하는 정확한 값을 읽을 수 있다.

그러나 `volatile` 변수에 셋팅할 새 값이 이 변수의 이전 값을 필요로 할 때 문제가 발생한다. `volatile` 변수를 읽고 새 값을 쓰는 사이에 짧은 `Race Condition`이 발생한다. (여러 스레드가 `volatile` 변수의 값을 똑같이 읽고, 새 값을 생성하여 이를 메인 메모리로 저장하는동안 서로의 값을 덮어쓸 수 있다.)

다음 테스트 코드를 보자

```java
public class VolatileTest {

    private static volatile long count = 0;

    @Test
    public void threadNotSafe() throws Exception {
        int maxCnt = 1000;

        for (int i = 0; i < maxCnt; i++) {
            new Thread(() -> count++).start();
        }

        Thread.sleep(100); // 모든 스레드가 종료될때 까지 잠깐 대기
        assertThat(count).isEqualTo(maxCnt);
    }
}
```

테스트 결과를 보면,
![images](https://1drv.ms/i/c/9251ef56e0951664/IQRMm0q_0WL7SomlSZF-oCo-AXjC0bMzqiuLxHnxkNYAe7w)

`count++` 연산은 `volatile` 변수가 가지는 이전 값에 의존하며, <u>원자성이 보장되지 않기 때문</u>에 동시성 문제는 동일하게 발생한다. `volatile`은 단지 모든 스레드가 캐시 없이 최신 값을 보게 할 뿐이다.

### 언제 사용하면 안 될까?
다음과 같은 상황에서는 `volatile` 대신 `synchronized`, `ReentrantLock` 또는 `Atomic` 변수를 사용해야 한다.

- 연산이 원자젹이지 않을 때 (Read-Modify-Write)
    - 변수의 이전 값에 의존하여 새로운 값을 만드는 모든 연산 (`count++`, `balance -= amount` 등)
- 둘 이상의 변수가 엮여 있을 때
    - 여러 변수가 함께 업데이트되어야 상태 일관성이 유지되는 경우
- 락(Lock)이 필요한 경우
    - 특정 코드 블록 전체에 대한 독점적인 접근이 필요한 경우
    - 변수를 읽고 쓸 때 `volatile`은 변수에 접근하는 다른 스레드를 블로킹하지 않는다.
- 읽기 스레드와 쓰기 스레드가 N:N인 경우
    - `volatile`은 단순히 하나의 스레드가 쓰고, 다른 스레드들은 읽기만 하는 상황(읽기 스레드와 쓰기 스레드가 N:1)과 같이 가시성만 보장되면 충분한 매우 제한적인 상황에서만 사용해야 한다. 읽는 스레드들은 언제나 이 변수의 가장 최근 수정된 값을 봐야하고, `volatile`은 이를 보장해 주기 떄문이다.


## References
- https://steady-coding.tistory.com/555
- https://dalichoi.tistory.com/entry/Lock-Free-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-%EC%82%B4%ED%8E%B4%EB%B3%B4%EA%B8%B0CAS-Volatile-Java-Atomic-Variables
- https://mong-dev.tistory.com/23
- https://shuu.tistory.com/57
- https://ttl-blog.tistory.com/238
- https://drg2524.tistory.com/195
- gpt4o