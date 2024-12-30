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

> e.g. 이진 세마포어를 사용하여 1로 초기화한 경우 `wait()`을 수행한 뒤 `signal()`을 수행해야 하는 일련의 순서를 지켜야 한다. 이 순서를 지키지 않으면 두 프로세스가 동시에 임계 영역에 접근하게 되어 경쟁 상태 같은 문제가 발생한다.

이러한 문제를 해결하기 위해 동기화 도구를 하이 레벨의 언어로 좀 더 편리하게 사용할 수 있도록 모니터 기법을 제공한다.

### 무엇인가
상호 배제와 조건의 동기화를 포함하는 고 수준의 동기화 구조

뮤텍스, 세마포어보다 좀 더 고 수준(high level, 추상화된)의 동시성 제어 기법. 프로그래밍 언어로 세마포어를 좀 더 편리하게 사용할 수 있도록 인터페이스를 제공하여 동시성 제어에 대한 개발자의 부담을 덜어준다.

시스템 콜과 유사한 개념. 시스템 콜은 사용자로부터 커널을 보호하기 위한 인터페이스.  커피 머신을 사용자가 직접 만지면 고장날 가능성이 높은 것처럼, OS가 관리하는 자원을 사용자가 마음대로 사용하게 두면 실수로 시스템 자원을 손상시킬 수 있다. 따라서 시스템 자원을 사용자로부터 숨기고 사용자의 요구사항을 처리할 수 있는 인터페이스만 제공하는데 이를 시스템 콜이라 했다.

시스템 콜처럼 모니터도 보호할 자원을 Critical Section으로 만들고 Critical Section에서 수행할 수 있는 인터페이스만 제공하여 자원을 보호한다.

모니터 내부에 공유 데이터와 공유 데이터에 접근하는 코드를 모두 정의한다. 그래서 공유 데이터에 접근하려면 모니터 안에 정의된 코드를 통해서만 접근할 수 있으며, 정의된 코드가 공유 데이터에 동시에 접근하면 모니터가 애초에 막는다. 그래서 개발자는 락을 걸고 푸는 코드를 추가할 필요가 없다. 모니터가 알아서 제어해 준다.

모니터에서 사용하는 상호 배제 방식은 뮤텍스를 의미한다.

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
- 모니터 타입만으로 동기화를 구현하는 데 충분하지 않으므로 `condition` 변수를 통해 동기화 매커니즘을 제공한다.
```
condition x, y;
x.wait();
x.signal();
```
- condition 변수에 호출되는 연산은 wait(), signal() 메소드
	- `x.wait()`를 호출한 프로세스는 다른 프로세스가 `x.signal()`을 호출할 때까지 대기
	- `x.signal()`는 정확히 대기중인 프로세스를 재개시킨다.


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

#### EntrySet
- 모니터 Lock을 획득하기 위해 대기 중인 스레드를 모아 놓은 자료 구조
- 스레드가 Lock을 사용 중인 경우, 다른 스레드는 EntrySet에 들어간다.
- EntrySet에 있는 스레드들은 Lock이 반납될 때까지 기다리며 락이 반납되면 EntrySet 중 한 스레드가 락을 획득하고 임계 영역으로 진입한다.

#### WaitSet
- `wait()` 호출한 스레드들이 대기하는 자료 구조
- 특정 조건이 만족할 때까지 대기할 때 WaitSet에 들어간다..
- Lock을 사용하던 스레드가 WaitSet에 들어가 대기할 때 Lock은 해제된다.
- 다른 스레드가 해당 객체의 `notify(), notifyAll()`을 호출하면 EntrySet으로 이동해서 다시 Lock을 획득할 수 있다.

### 협력(Cooperation) - wait()과 notify(), Condition Variable
`synchronized` 블록에서 사용되는 `java.lang.Object` 클래스의 `wait(), notify(), notifyAll()` 메소드와 모니터의 조건 변수(Condition Variable)를 통해 스레드 간 협력을 가능하게 한다.

A 스레드가 특정 조건을 만족하지 못해 대기하면 B 스레드가 특정 조건을 만족시킨 뒤 A 스레드를 깨우는 방식으로 협력한다.

- Java 모니터는 오직 한 개의 조건 변수만 가질 수 있다.
- 조건 변수는 `wait(), notify(), notifyAll()` 과 함께 작동하며 특정 조건이 충족될 때까지 다른 스레드를 대기시킬 수 있다.
- 스레드가 특정 조건에 충족되지 않으면 `wait()` 메서드를 호출해 조건 변수의 대기 셋(wait-set)에서 대기한다. (대기 상태로 전환)
- 스레드가 특정 조건을 만족해서 `notify()` 또는 `notifyAll()` 메서드를 호출해 해당 조건 변수의 대기 셋으로부터 스레드를 깨운다.
- 조건 변수를 통해 스레드 간 `wait()`과 `notify()`를 조절하면서 경쟁 조건(Race Condition) 같은 문제를 방지할 수 있다.
- `wait(), notify(), notifyAll()`은 모두 `synchronized` 블록 안에서만 사용해야 한다. (즉, 모니터 락을 확보한 상태에서만 동작한다.)

어떤 스레드가 모니터를 먼저 소유할지에 따라 조건 변수의 종류 Signal and Wait, Signal and Continue 두 가지로 구분할 수 있다. (Java에서는 `Signal and Continue` 방식을 취한다.)

#### 조건 변수의 종류
##### Signal and Wait
- 현재 모니터를 소유한 스레드가 wait()을 실행하면 자신을 일시 중단하고 Lock을 해제한 후 Wait Set에 들어간다.
- 깨우는 스레드가 `notify()` 또는 `notifyAll()`을 호출하면 Wait Set에 있는 대기 중인 스레드를 깨우고 해당 스레드는 Lock을 해제하고 대기한다.
- 대기에서 깨어난 스레드가 Lock을 획득한 후 모든 작업을 마치고 Lock을 해제하면 깨운 스레드가 Lock을 획득한 후 작업을 계속 진행한다.
- 대기 스레드와 깨운 스레드 사이에 다른 스레드가 모니터를 소유할 수 없도록 원자적 실행이 보장되어야 한다.

##### Signal and Continue
- 현재 모니터를 소유한 스레드가 wait()을 실행하면 자신을 일시 중단하고 Lock을 해제한 후 Wait Set에 들어간다.
- 깨우는 스레드가 `notify()` 또는 `notifyAll()`을 호출하면 Wait Set에 있는 대기 중인 스레드를 깨운다. 이 때 일어난 스레드들은 Entry Set으로 이동한다.
- 깨우는 스레드는 Lock을 계속 유지하면서 모든 작업을 완료하고 Lock을 해제하면 Entry Set에 대기하고 있는 모든 스레드가 Lock을 획득하기 위해 경쟁한다.

### Synchronized
- 명시적인이지 않은 Java에 내장된 락으로 암묵적인 락 `Intrinsic Lock`, 모니터 락 `Monitor Lock` 이라고도 한다.
- 임계 영역으로 지정할 영역을 선언할 때 사용하는 Java 키워드
- `synchronized` 블록은 해당 객체의 모니터를 획득할 수 있으며, 모니터를 획득한 단 하나의 스레드만 임계 영역에 접근 가능하고 그 외 다른 스레드들은 차단되어 대기 상태가 된다.
- `synchronized` 블록을 빠져나오면 모니터 락이 해제되고, 대기 중인 다른 스레드 중 하나가 모니터 락을 얻고 임계 영역에 진입하여 작업을 수행하는 방식으로 상호 배제가 보장된다.
- `synchronized`가 적용된 하나의 메서드만 호출해도 같은 모니터의 모든 synchronized 메서드까지 락에 잠기게 되어 락이 해제될 때까지 접근 불가하다.
- `sleep()`을 실행한 스레드는 동기화 영역에서 대기 중이더라도 획득한 락을 놓거나 해제하지 않는다.
- `synchronized`의 동기화 영역에 진입하지 못하고 대기 중인 스레드는 인터럽트되지 않는다.

- 메서드나 코드 블록에 적용 가능
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
- 인스턴스 단위로 모니터가 동작하며 동일한 인스턴스 안에서 `synchronized`가 적용된 곳은 하나의 락을 공유한다.
- 인스턴스가 여러 개일 경우 인스턴스별로 모니터 별로 락을 획득해서 동기화 영역에 진입하고 빠져나올 때 락을 해제할 수 있다.

```java
public class MyClass {
    public synchronized void syncMethod1() {
        // Critical Section
    }
    public synchronized void syncMethod2() {
        // Critical Section
    }
    
    public static void main(String[] args) {
        // 동기화에 대해서 서로 영향이 없다.
        MyClass m1 = new MyClass().syncMethod1();
        MyClass m2 = new MyClass().syncMethod1();
    }
}
```

**[정적 메서드 동기화]**
- 클래스 단위로 모니터가 동작하며 `synchronized`가 적용된 곳은 하나의 락을 공유한다.
- 인스턴스와는 별개의 모니터를 가지고 임계 영역을 동기화하기 때문에 인스턴스 단위로 메서드를 호출할지라도 클래스 단위로 스레드간 공유한다.
- 클래스는 메모리에 오직 하나만 존재하므로 하나의 모니터를 공유해서 동기화하고자 할 때 사용할 수 있다.

```java
public class MyClass {
    public synchronized void syncMethod1() {
        // Critical Section
    }
    public synchronized void syncMethod2() {
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

#### 동작 방식
![monitor](https://1drv.ms/i/c/9251ef56e0951664/IQREguO9djAtSK0FE5vM3eOHAToC4B6MM5bzTySUrnlQYw8)
TODO...


#### 재진입성
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

Sub sub = new Sub();
sub.method2();
```

- 재진입 가능하다는 것은 락의 획득이 호출 단위가 아닌 스레드 단위로 일어난다는 것을 의미하며 이미 락을 획득한 스레드는 같은 락을 얻기 위해 <u>대기할 필요 없이 `synchronized` 블록을 만났을 때 같은 락을 확보하고 진입</u>한다.

#### 상속
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

#### 가시성
- 한 스레드가 공유 자원에 수정, 쓰기 작업할 때 다른 스레드가 수정한 내용이 보이는 것을 '가시성' 이라 한다.
- `synchronized`는 가시성을 지원한다.
	- CPU 캐시에 수정한 변수 값을 쓰지 않고 메모리에 직접 쓰는 것을 의미
	- 멀티 스레드 환경에서 작업 시 CPU가 참조하는 메모리가 달라 데이터 불일치 방지

#### 비공정성
- `synchronized` 영역에 진입하지 못한 다른 스레드가 모니터를 획득하는 데 순서가 정해져 있지 않다.
- 기아 상태에 빠진 스레드가 나올 수 있지만 OS 레벨에서 적절히 처리한다.

## References
- https://www.baeldung.com/cs/monitor
- https://devdebin.tistory.com/16
- https://devdebin.tistory.com/335
- https://velog.io/@bbamjoong/xuhnwflw
- https://velog.io/@appti/%EC%9E%90%EB%B0%94-synchronized
- https://howudong.tistory.com/339
- https://mini98.tistory.com/71
- Silberschatz et al. 『Operating System Concepts』. WILEY, 2020.
- gpt4o