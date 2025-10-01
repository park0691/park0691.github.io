---
title: Java 동시성 제어 기법 (5) - Synchronized Collection, Concurrent Collection
date: 2025-09-22 16:00 +0900
categories: Java
tags: concurrent, synchronized, collections, concurrenthashmap
---

## Synchronized Collection
### 특징
**[동기화 방식: 컬렉션 전체에 단일 락]**<br/>
Synchronized 컬렉션은 `add`, `get`, `remove` 등 데이터에 접근하는 모든 메서드에 `synchronized`키워드를 사용한다. 이는 컬렉션 객체 전체를 단 하나의 락(Lock)으로 제어하는 **'전체 잠금(Single Global Lock)' 방식**으로, 열쇠가 하나뿐인 방과 같다.

결과적으로 한 번에 오직 하나의 스레드만 접근할 수 있으므로, 한 스레드가 값을 읽는 동안 다른 스레드는 값을 추가/삭제하는 것은 물론, 다른 데이터를 읽는 작업조차 할 수 없다.

**[Fail-Fast: 빠른 실패]**<br/>
`Iterator`를 이용한 컬렉션 순회 도중 다른 스레드가 컬렉션을 수정하면 `ConcurrentModificationException` 예외를 발생시키고 작업을 중단한다.

즉, 순회 도중에 데이터가 변경되는 것을 **'심각한 오류'**이자 **'예측 불가능한 상태'**로 간주하는 것이다. 일관성이 깨진 데이터로 계속 작업을 진행하면, 결과가 완전히 달라지거나 더 큰 문제를 일으킬 수 있다고 본다. 문제 발생의 여지가 있다면 즉시 실패 처리하는 전략으로, 의도하지 않은 문제 발생을 원천 차단할 수 있다.

- 동작 방식
    - 반복자 생성 시점에 내부에 `modCount` 같은 변경 횟수를 기록하는 값을 생성 시점의 컬렉션의 실제 변경 횟수로 초기화한다. `next()` 메서드 호출할 때마다 `Iterator`가 기억한 변경 횟수와 컬렉션의 실제 변경 횟수를 비교하여 값이 다르면 다른 스레드의 개입이 있었다 판단하고 예외를 발생시킨다.

- 문제점
    - 단일 스레드 환경에서 데이터 불일치를 빠르게 감지할 수 있지만, 멀티 스레드 환경에서는 예외를 발생시켜 프로그램이 멈출 수 있다. 순회 중에는 컬렉션을 수정하지 않도록 해야 한다.


### 종류
#### Vector
```java
package java.util;

public class Vector<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    ...
    public synchronized int capacity() {
        return elementData.length;
    }

    public synchronized int size() {
        return elementCount;
    }

    public synchronized boolean isEmpty() {
        return elementCount == 0;
    }
    ...
    public synchronized E get(int index) {
        if (index >= elementCount)
            throw new ArrayIndexOutOfBoundsException(index);

        return elementData(index);
    }
    ...
    public synchronized boolean add(E e) {
        modCount++;
        ensureCapacityHelper(elementCount + 1);
        elementData[elementCount++] = e;
        return true;
    }
    ...
    public synchronized E remove(int index) {
        modCount++;
        if (index >= elementCount)
            throw new ArrayIndexOutOfBoundsException(index);
        ...
        return oldValue;
    }
}
```
메서드 단위로 `synchronized` 키워드가 보인다. 컬렉션 내부를 수정하는 메서드뿐만 아니라 조회하는 메서드에도 동기화 처리가 되어있음을 확인할 수 있다.

#### Hashtable
```java
package java.util;

public class Hashtable<K,V>
    extends Dictionary<K,V>
    implements Map<K,V>, Cloneable, java.io.Serializable {
    ...
    public synchronized int size() {
        return count;
    }

    public synchronized boolean isEmpty() {
        return count == 0;
    }
    ...
    public synchronized V get(Object key) {
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        ...
        return null;
    }
    ...
    public synchronized V put(K key, V value) {
        ...
        addEntry(hash, key, value, index);
        return null;
    }
    
    public synchronized V remove(Object key) {
        Entry<?,?> tab[] = table;
        ...
        return null;
    }
    ...
}
```
Hashtable도 메서드 단위로 `synchronized` 키워드가 보인다. 컬렉션 내부를 수정하는 메서드뿐만 아니라 조회하는 메서드에도 동기화 처리가 되어있음을 확인할 수 있다.

#### Collections.synchronizedXXX()
유틸리티 클래스인 `java.util.Collections`의 `static inner` 클래스이다. Hashtable 처럼 레거시 클래스는 아니며 JDK 1.5 버전에 도입되었다.

```java
package java.util;

public class Collections {
    ...
    public static <T> List<T> synchronizedList(List<T> list) {
        return (list instanceof RandomAccess ?
                new SynchronizedRandomAccessList<>(list) :
                new SynchronizedList<>(list));
    }
    ...
    public static <T> Set<T> synchronizedSet(Set<T> s) {
        return new SynchronizedSet<>(s);
    }
    ...
    public static <K,V> Map<K,V> synchronizedMap(Map<K,V> m) {
        return new SynchronizedMap<>(m);
    }
    ...
```

`Collections.synchronizedList()`나 `Collections.synchronizedMap()`과 정적 팩토리 메서드를 통해 스레드 안전하지 않은 컬렉션을 스레드 안전한 컬렉션으로 변환해 준다.

```java
package java.util;

public class Collections {
    ...
    static class SynchronizedList<E>
        extends SynchronizedCollection<E>
        implements List<E> {
        private static final long serialVersionUID = -7754090372962971524L;

        final List<E> list;
        ...
        public int hashCode() {
            synchronized (mutex) {return list.hashCode();}
        }

        public E get(int index) {
            synchronized (mutex) {return list.get(index);}
        }
        public E set(int index, E element) {
            synchronized (mutex) {return list.set(index, element);}
        }
        public void add(int index, E element) {
            synchronized (mutex) {list.add(index, element);}
        }
        public E remove(int index) {
            synchronized (mutex) {return list.remove(index);}
        }
    ...
}

```

`SynchronizedList` 내부를 보면 `get`, `set`, `add`등 모든 메서드에 `synchronized` 블록이 적용되어 있으며, 하나의 `mutex`를 공유하는 것을 확인할 수 있다.

모든 읽기, 쓰기에 동기화되어 있으므로 융통성이 없는 설계다.


### 한계
**[성능 저하]**<br/>
Synchronized 컬렉션의 가장 큰 한계는 컬렉션 전체를 단 하나의 락(Lock)으로 제어하는 **'전체 잠금'** 방식에서 비롯되는 성능 저하이다.

이 방식은 내부 데이터에 대한 모든 접근(읽기/쓰기 등)을 순차적으로 만들어 스레드 안전성을 보장하지만, 읽기 작업과 쓰기 작업이 서로 블로킹되어 동시에 많은 양을 처리할 수 없었고, 컬렉션 단위로 락이 잡혀 동시 처리량이 크게 저하된다.


## Concurrent Collection
Java 5의 `java.util.concurrent` 패키지를 통해 도입된 고성능 동시성 컬렉션이다. 기존 Synchronized 컬렉션의 성능 한계를 극복하기 위해, 여러 스레드의 동시 접근을 허용하는 정교한 동기화 설계를 반영하여 월등히 뛰어난 성능과 확장성을 제공한다.

### 주요 구현체
![images](https://1drv.ms/i/c/9251ef56e0951664/IQRWQFvmqoPaQ4B2s6Q8HfEDAa3qzGrHI9Wyk4rLN8C23Wg)
_출처 : https://www.logicbig.com/tutorials/core-java-tutorial/java-collections/concurrent-collection-cheatsheet.html_

### 특징
**[동기화 방식: 세분화된 락]**<br/>
Concurrent 컬렉션은 컬렉션 전체를 하나의 락으로 막는 '전체 잠금' 방식 대신, 더 정교하고 세분화된 동기화 기법을 사용하여 성능을 극대화한다. 주요 전략으로는 락 스트라이핑, Lock-Free, 그리고 Copy-on-Write가 있습니다.

- **Lock Striping** : 데이터의 특정 부분에만 락을 거는 기법 (예: `ConcurrentHashMap`)
- **Lock-Free (CAS)** : 락 없이 원자적 연산을 사용하는 기법 (예: `ConcurrentLinkedQueue`)
- **Copy-on-Write** : 데이터를 수정할 때마다 복사본을 만들어 동시성을 보장하는 기법 (예: `CopyOnWriteArrayList`)

이러한 세분화된 기법들 덕분에 여러 스레드가 컬렉션에 동시에 접근하여 작업을 수행할 수 있다. 특히 데이터를 변경하지 않는 읽기(read) 작업은 대부분의 경우 락 없이 동시에 수행되며, 쓰기(write) 작업 또한 데이터의 일부 영역만 잠그므로 다른 영역에 대한 접근을 방해하지 않아 높은 처리량을 보장한다.

**[Fail-Safe: 안전한 실패]**<br/>
`Iterator`를 이용한 컬렉션 순회 도중 다른 스레드가 컬렉션을 수정하더라도 **예외를 발생시키지 않고** 작업을 계속 진행하는 방식이다.

이 방식은 순회 도중에 데이터가 변경될 수 있다는 사실을 **'자연스러운 현상'**으로 받아들인다. 예외를 발생시켜 전체 작업을 중단시키는 것보다, 일단 현재 진행 중인 순회 작업이라도 안정적으로 마치는 것이 더 중요하다고 본다.

- **Copy-on-Write (`CopyOnWriteArrayList`)** : 반복자가 생성될 때의 내부 데이터 복사본(스냅샷)을 가지고 순회한다. 따라서 원본 데이터를 수정하더라도 반복자는 영향 받지 않고 생성 시점의 데이터를 안전하게 순회한다.
- **Weakly Consistent (`ConcurrentHashMap`)** : 추가/제거 메소드에 `synchronized` 키워드를 사용하여 락을 잡아 스레드 안전을 보장한다. 반복자 순회시에는 원본 컬렉션을 순회하므로, 순회를 시작한 이후에 삽입된 데이터는 노출되거나 노출되지 않을 수 있지만 예외가 발생하지 않는다.

멀티 스레드 환경에서 컬렉션을 순회하는 동안에도 다른 스레드가 안전하게 데이터를 수정할 수 있어 동시성이 높고 안정적이다.

- 안정성 : `ConcurrentModificationException`이 발생하지 않아 프로그램이 중단될 위험이 없다.
- 비일관성 : `Iterator`는 순회 시작 시점의 데이터를 보기 때문에, 순회 도중 다른 스레드에 의해 추가되거나 변경된 최신 데이터를 반영하지 못할 수 있다.


### 종류
이 포스트에서는 대표적인 몇 가지 컬렉션만 간단히 다룬다.

#### ConcurrentHashMap (ConcurrentMap 구현체)
`ConcurrentMap` 인터페이스의 구현체로 `Hashtable`, `Collections.synchronizedMap`을 대체하기 위해 도입된 고성능 스레드 안전 Map이다. Map 전체를 하나의 락으로 제어하는 대신 세분화된 잠금(Fine-Grained Locking) 기법을 적용하여, 여러 스레드의 동시 접근을 허용함으로써 월등히 뛰어난 처리량과 확장성을 제공한다.

> **ConcurrentMap**<br/>
> 기존 Map 인터페이스에 **원자적 연산**을 추가하여 멀티 스레드 환경에서 안전하게 사용할 수 있도록 확장한 인터페이스이다.<br/>
> Map을 synchronizedMap으로 감싸도 특정 조건에 따라 값을 추가하거나 변경하는 복합 연산에서는 여전히 경쟁 상태가 발생할 수 있다. 다음 코드를 보자.
> 
> ```java
> if (!map.containsKey(key)) {
>    map.put(key, value); // 이 두 작업 사이에 다른 스레드가 끼어들 수 있음!
> }
> ```
> 
> 키가 없을 때만 추가하는 위 코드에서, `containsKey()`와 `put()` 각각은 동기화될 수 있지만, 두 메서드를 합친 전체 작업의 원자성은 보장되지 않는다.<br/>
> ConcurrentMap은 이러한 이런 복합 동작을 하나의 원자적 연산으로 처리할 수 있는 새로운 메서드를 제공하여 문제를 해결한다.
> 
> **[주요 원자적 메서드]**<br/>
> 
> | 메서드 | 설명 |
> | ------ | ------ |
> | putIfAbsent(K key, V value) | 맵에 key가 존재하지 않을 경우에만 값을 추가한다.<br/>containsKey와 put을 합친 원자적 연산이다. | 
> | remove(Object key, Object value) | key가 특정 value와 매핑되어 있을 경우에만 항목을 삭제한다. |
> | replace(K key, V oldValue, V newValue) | key가 특정 oldValue와 매핑되어 있을 경우에만 newValue로 교체한다. |
{: .prompt-tip }
>


**[Hashtable과 synchronizedMap의 공통적인 문제: 전체 잠금]**<br/>
`Hashtable`과 `Collections.synchronizedMap`은 멀티스레드 환경에서 안전성을 보장하기 위해, 데이터에 접근하는 모든 메서드(get, put, remove 등)를 synchronized로 묶는다.

이는 Map 객체 전체에 단 하나의 '열쇠(Lock)'만 존재하는 것과 같다. 어떤 스레드가 Map에 접근하면, 다른 스레드들은 그 스레드가 작업을 마칠 때까지 읽기, 쓰기 등 모든 작업을 멈추고 기다려야만 한다.

이 방식은 다음과 같은 심각한 문제를 유발한다.

- 성능 병목 현상 : 모든 스레드가 단 하나의 락을 얻기 위해 경쟁하므로, 동시 사용자가 많아질수록 성능이 급격히 저하된다.

- 불필요한 대기 : 데이터를 읽기만 하는 작업조차 다른 읽기/쓰기 작업을 모두 차단하여 시스템의 전체 처리량을 떨어뜨린다.


**[ConcurrentHashMap: 세분화된 잠금]**<br/>
![images](https://1drv.ms/i/c/9251ef56e0951664/IQSIOtTqNU64QbJqVveEP7NKAXi1mGbHikDaJQPcGRxE6dI?width=960&height=486)
_출처 : https://codepumpkin.com/hashtable-vs-synchronizedmap-vs-concurrenthashmap/_

`ConcurrentHashMap`은 락의 범위를 전체 Map이 아닌 데이터의 일부 영역으로 한정하여, 여러 스레드가 서로 다른 데이터 영역에서 동시에 작업할 수 있도록 허용한다.

이러한 설계는 락의 범위를 극도로 세분화하여 여러 스레드가 동시에 Map을 수정할 수 있도록 동시성을 높인 `ConcurrentHashMap`의 성능 향상 비결이라 볼 수 있다.

- **(Java 7 이전)** : 데이터를 `ReentrantLock`을 상속받은 **세그먼트(Segment)**로 나누고, 각 세그먼트마다 별도의 락을 두었다.

- **(Java 8 이후)** : 락의 단위를 버킷(Bucket)의 첫 노드 수준까지 극도로 세분화했다.
    - 빈 버킷에 노드를 삽입할 경우 CAS 연산을 사용해 원자적으로 첫 번째 노드를 저장한다.
    - 삽입할 버킷에 이미 다른 노드가 존재한다면(해시 충돌), 락을 사용한다. 이 락은 Map 전체나 큰 단위의 세그먼트가 아닌, 해당 버킷의 첫 번째 노드(head)를 기준으로 `synchronized` 블록으로 잠근다.
    - 이 덕분에 각 테이블 버킷을 독립적으로 잠글 수 있으며 한 버킷에서 락 경합이 발생하더라도 다른 버킷에 접근하는 스레드들은 전혀 영향을 받지 않는다.


#### CopyOnWriteArrayList
읽기 작업이 쓰기 작업보다 많을 때 최고의 성능을 발휘하는 List 구현체

```java
package java.util.concurrent;

public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    ...
    final transient ReentrantLock lock = new ReentrantLock();

    private transient volatile Object[] array;
    ...
    public E get(int index) {
        return elementAt(getArray(), index);
    }
    ...
    public void add(int index, E element) {
        synchronized (lock) {
            Object[] es = getArray();
            ...
            Object[] newElements;
            int numMoved = len - index;
            if (numMoved == 0)
                newElements = Arrays.copyOf(es, len + 1);
            else {
                newElements = new Object[len + 1];
                System.arraycopy(es, 0, newElements, 0, index);
                System.arraycopy(es, index, newElements, index + 1,
                                 numMoved);
            }
            newElements[index] = element;
            setArray(newElements);
        }
    }
    ...
```

**Copy-on-Write(쓰기 시 복사)** : 데이터를 변경하는 작업(`add`, `set`, `remove`)이 발생할 때 원본 배열의 요소를 복사하여 새로운 임시 배열을 만들고, 임시 배열에 쓰기 동작을 수행한 후 원본 배열을 원자적으로 교체한다.

읽기 작업은 락을 걸지 않고 기존 데이터를 참조하므로 전혀 영향을 받지 않는다.

쓰기 작업 시 배열 복사 비용이 매우 높기 때문에 쓰기 작업이 많은 경우 성능 저하가 발생할 수 있다.

데이터 변경은 거의 일어나지 않지만 여러 스레드가 동시에 데이터를 읽는 환경에 적합하다.

**[주요 용도]**<br/>
쓰기 비용이 매우 비싸므로, 데이터 변경이 거의 일어나지 않지만 여러 스레드가 동시에 데이터를 읽는 환경(예: 이벤트 리스너 목록, 설정 정보)에 적합하다.

#### ConcurrentLinkedQueue
`Queue` 인터페이스 구현체

**Lock-Free** 방식으로 구현된 스레드 안전한 **FIFO** 방식 큐. 락을 전혀 사용하지 않고, CAS 연산만을 이용해 노드를 추가/제거한다. (*Non-blocking Queue*)

**[동작 방식]**<br/>
여러 스레드가 동시에 `offer`, `poll` 작업을 수행해도 CAS 연산을 통해 원자적으로 노드의 포인터를 변경하여 데이터의 일관성을 유지한다.

**[주요 용도]**<br/>
여러 생산자(Producer)와 소비자(Consumer) 스레드가 동시에 데이터를 주고받는 작업 큐(Queue)에 적합하다.

#### BlockingQueue 구현체
`BlockingQueue` 인터페이스 구현체로, 생산자-소비자(Producer-Consumer) 패턴 구현에 최적화되어 있다. 큐가 비어있을 때 데이터를 가져가려는 스레드나, 큐가 가득 찼을 때 데이터를 넣으려는 스레드를 **블로킹(blocking)** 시키는 기능이 내장되어 있다.

- ArrayBlockingQueue : 크기가 정해져 있는(bounded) 배열 기반의 블로킹 큐

- LinkedBlockingQueue : 크기를 지정할 수 있는(optionally-bounded) 링크드 리스트 기반 블로킹 큐. ArrayBlockingQueue보다 처리량이 높다.

- PriorityBlockingQueue : 우선순위가 있는 요소들을 처리하는 무한(unbounded) 블로킹 큐.

- SynchronousQueue : 크기가 0인 블로킹 큐. 한 스레드가 데이터를 넣으면, 다른 스레드가 그 데이터를 가져갈 때까지 블로킹된다. 스레드 간의 직접적인 데이터 교환에 사용된다.

ConcurrentLinkedQueue와 비슷하다. 그러나 큐가 비어 있다면 큐에서 항목을 뽑아내는 연산은 새로운 항목이 추가될 때까지 대기하고 반대로 큐에 크기가 지정된 경우에 큐가 지정한 크기만큼 가득 차 있다면, 큐에 새 항목을 추가하는 연산은 큐에 빈 자리가 생길 때까지 대기한다는 차이점이 있다.

#### ConcurrentSkipListMap, ConcurrentSkipListSet
SortedMap, SortedSet 클래스의 동시성을 높이도록 발전된 형태

데이터가 키 순서대로 정렬되는 것을 보장하는 Concurrent 컬렉션.

내부적으로 **스킵 리스트(Skip List)** 자료구조를 사용하여 정렬과 동시성을 모두 효율적으로 처리한다.

**[동작 방식]**<br/>
여러 레벨의 링크를 가진 리스트 구조를 사용하여 데이터를 검색/삽입할 때 락의 범위를 최소화하면서 빠르게 작업을 수행한다.

**[주요 용도]**<br/>
정렬된 상태를 유지해야 하는 스레드 안전한 Map, Set이 필요할 때 사용된다.

## References
- https://medium.com/@qwebnm9900/synchronized-concurrent-collections-b0e16d7f3eaa
- https://codepumpkin.com/hashtable-vs-synchronizedmap-vs-concurrenthashmap/
- https://steady-coding.tistory.com/575
- https://blog.hexabrain.net/403
- Gemini 2.5 Pro