---
title: ConcurrentHashMap 분석 - 데이터 조회, 저장 시 동시성 제어
date: 2025-10-20 23:30 +0900
categories: Java
tags: concurrent, collections, concurrenthashmap
---

`ConcurrentHashMap`은 Java에서 스레드 안전한 맵(Map)을 구현할 때 가장 먼저 고려되는 표준 컬렉션이다. Java 8에서 `ConcurrentHashMap`은 기존의 세그먼트(Segment) 락 방식에서 벗어나, **CAS(Compare-And-Swap) 연산**과 **버킷 레벨 락(Bucket-level Lock)**을 결합한 정교한 방식으로 재설계되어 극대화된 동시성을 제공한다.

이 포스트에서는 Java 8을 기준으로 ConcurrentHashMap이 어떻게 데이터를 **조회**하고 **저장**하는지, 그 내부 동작 원리를 소스 코드와 함께 심층 분석하고자 한다.

먼저, ConcurrentHashMap의 다양한 생성자를 살펴보며 `initialCapacity`와 `concurrencyLevel` 같은 파라미터가 어떤 의미 인지 알아본다.

다음으로 데이터 조회(`get()`) 메서드가 어떻게 락 없이 데이터를 빠르게 읽어오는지 확인한다. 이 과정에서 Node의 volatile 필드가 메모리 가시성을 보장하는 방식과, 이로 인해 발생하는 **"약한 일관성(Weak Consistency)"**의 의미를 짚어본다.

그리고 ConcurrentHashMap 동시성 제어의 핵심인 데이터 저장(`putVal()`) 메서드를 상세히 파헤친다.

마지막으로 코드를 분석하며 떠오른 **'왜 데이터를 저장할 때 CAS 연산을 버킷이 빈 경우에만 사용할까?'** 라는 의문에 대한 필자의 생각을 서술했다.

## 생성자

1. 디폴트 생성자
```java
ConcurrentHashMap()
```
- 내부적으로 `DEFAULT_CAPACITY` (16) 및 `DEFAULT_LOAD_FACTOR` (0.75)를 사용한다.

2. 생성자 [1]
```java
ConcurrentHashMap(int initialCapacity)
```
- 맵이 수용할 수 있는 해시 테이블 최소 크기 지정
- 내부적으로 이 용량을 기준으로 해시 테이블 크기(2의 거듭제곱)가 설정된다.

3. 생성자 [2]
```java
ConcurrentHashMap(Map<? extends K, ? extends V> m)
```
- 주어진 맵의 모든 맵핑을 포함하는 `ConcurrentHashMap`을 생성한다.

4. 생성자 [3]
```java
ConcurrentHashMap(int initialCapacity, float loadFactor)
```
- 초기 용량과 로드 팩터를 지정하여 `ConcurrentHashMap`을 생성한다.

5. 생성자 [4]
```java
ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel)
```
- 세 가지 파라미터를 모두 지정한다.

    ```java
    public ConcurrentHashMap(int initialCapacity,
                                 float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (initialCapacity < concurrencyLevel)   // Use at least as many bins
            initialCapacity = concurrencyLevel;   // as estimated threads
        ...
    ```

    - 하지만 Java 8 이후 `ConcurrentHashMap`에서 `concurrencyLevel` 파라미터는 무시되며, 단지 초기 테이블 크기를 결정하는 힌트로만 사용된다.

    > **주요 파라미터**<br/>
    > - `initialCapacity`: 맵이 생성될 때 할당되는 내부 해시 테이블 배열의 크기. 충분히 큰 값으로 설정하면 빈번한 초기 크기 조절(resizing)을 방지하여 성능을 향상시킬 수 있다.
    > - `loadFactor`: ConcurrentHashMap에서는 로드 팩터가 기본적으로 0.75로 고정됨. 즉, 테이블 용량의 75%가 채워지면 크기 조절이 시작된다. 생성자에서 로드 팩터를 다르게 지정하더라도 내부 동작은 0.75를 따른다.
    > - `concurrencyLevel`: 맵을 동시에 업데이트하는 스레드의 수. Java 7까지는 맵의 동시성 제어를 위한 세그먼트의 최대 개수(즉, 락의 수)를 결정하므로 동시성 제어의 핵심 요소였으나, Java 8부터 전체 맵에 걸친 동적 크기 조절 및 CAS 기반 동기화로 변경되어, 이 파라미터는 더 이상 락의 개수 결정에 영향을 미치지 않는다. 단지 초기 해시 테이블의 크기를 계산하는 힌트로만 사용된다.
    >
    {: .prompt-tip }

## 데이터 조회
### get()
`get()` 메소드는 락 없이 키에 해당하는 값을 빠르게 조회하기 위해 설계되었다.<br/>
다음 코드를 보면 `synchronized`나 CAS 연산이 사용되지 않는 것을 알 수 있다. 

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        // 첫 번째 노드 직접 비교
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 특수(제어) 노드 처리
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        // 연결 리스트(체인) 순회
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

**[약한 일관성, Weak Consistency]**<br/>
`ConcurrentHashMap`은 데이터 조회 시 동시성 문제를 일으키지 않지만 최신 데이터를 반드시 보장하지 않는다. 한 스레드가 데이터를 쓰는 도중 다른 스레드가 데이터를 수정하면, 일부만 갱신된 상태의 데이터나 수정 전의 값을 읽을 수 있다.

`ConcurrentHashMap`은 데이터 일관성보다 읽기 성능 최적화에 중점을 두고 있기 때문에 이러한 약한 일관성 전략을 채택하여 읽기 작업에 대한 락 경합을 완전히 제거한 것 같다.

이러한 방식은  데이터의 즉각적인 일관성이 필수적이지 않고 **높은 처리량(Throughput)**이 중요한 환경(예: 카운터, 캐시)에서 매우 효과적이다.

**[메모리 가시성 보장]**<br/>
그럼에도 `get()`이 동시성을 보장하는 이유는 내부적으로 `volatile` 변수를 사용해 메모리 가시성을 보장하기 때문이다. `ConcurrentHashMap` 버킷들은 `Node` 배열을 사용하고, 이 배열에 대한 참조는 `volatile` 로 선언되어 있다. 이를 통해서 여러 스레드가 동시에 읽기를 할 때 최신 값을 읽을 수 있도록 보장된다. 


> **HashMap과 다른 Node 차이점**<br/>
> ```java
> static class Node<K,V> implements Map.Entry<K,V> {
>     final int hash;
>     final K key;
>     volatile V val;
>     volatile Node<K,V> next;
>     ...
> ```
> 
> 1. volatile 사용<br/>
> 기존 HashMap와 달리 `val`, `next` 필드에 `volatile` 키워드가 사용된다.<br/>
> `ConcurrentHashMap`은 버킷에 새로운 노드를 삽입할 때 CAS 연산을 통해 락 없이 원자성을 확보한다. 하지만 노드의 **내부 필드**에 대한 변경의 가시성을 보장하기 위해 `volatile`을 사용하여 동시 읽기/쓰기 상황에서 데이터 정합성을 유지한다.
> 
> 2. 해시 필드의 용도 확장<br/>
> `ConcurrentHashMap`의 `Node`는 `HashMap`에 없는 특수 목적의 **제어 노드**를 포함하며, 이들은 음수 해시 값을 가진다. 자세한 내용은 `putVal()` 메소드의 `fh` 변수에 대해 작성한 부분을 참고하자.
> 
{: .prompt-tip }


## 데이터 저장
`ConcurrentHashMap`은 높은 동시성을 확보하기 위해 CAS(Compare-And-Swap) 연산과 버킷 레벨 동기화를 활용하여 스레드 안전한 업데이트를 수행한다.

구체적으로, 빈 버킷에 새로운 노드를 삽입할 때는 락 없이 **CAS 연산**을 사용한다. 반면 이미 노드가 존재하는 버킷에서는 맵 전체에 락을 거는 대신, 해당 **버킷의 첫 번째 노드**를 기준으로 부분적인 락을 획득하여 스레드 경합을 최소화한다.

어떻게 스레드 안전을 구현하는지 코드를 통해 살펴보자

### put(), putIfAbsent()
```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
```

```java
public V putIfAbsent(K key, V value) {
    return putVal(key, value, true);
}
```
엔트리를 추가, 업데이트할 때 주로 사용하는 메서드는 `put()`, `putIfAbsent()`다. 두 메서드는 내부적으로 `putVal()` 메서드를 호출한다.

둘의 차이는 키가 이미 존재하는 경우 기존 값을 그대로 유지할지인데, 이는 `putVal()` 메서드 호출 시 **세 번째 파라미터 `onlyIfAbsent`**로 전달한다.

### putVal()
```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
        else if (onlyIfAbsent // check first node without acquiring lock
                 && fh == hash
                 && ((fk = f.key) == key || (fk != null && key.equals(fk)))
                 && (fv = f.val) != null)
            return fv;
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                    else if (f instanceof ReservationNode)
                        throw new IllegalStateException("Recursive update");
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```
- 무한 루프 구조로, 작업이 완료됬을 때 `break`문으로 빠져나오는 방식으로 되어 있다.
- `onlyIfAbsent` 파라미터 : 값이 `true`면 키가 없을 때만 값을 삽입한다.

#### 해시 테이블 초기화
```java
if (tab == null || (n = tab.length) == 0)
    tab = initTable();
```

해시 테이블이 초기화되지 않았을 때(또는 테이블 사이즈가 0), 다음 `initTable()` 메서드가 호출된다.

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

- 여러 스레드가 동시에 `put` 또는 `get`메서드를 호출하면서 `initTable()`을 수행할 수 있다. 이러한 초기 스레드 경합 발생을 방지하기 위해 6L(6번째 줄) CAS 연산을 수행하는 것을 확인할 수 있다.

#### 노드 추가: 버킷이 빈 경우 (CAS)
```java
else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
    if (casTabAt(tab, i, null,
                 new Node<K,V>(hash, key, value, null)))
        break;                   // no lock when adding to empty bin
```

- 저장될 데이터 인덱스 `i`의 해시 버킷이 비었다면 CAS 연산으로(`casTabAt()` 호출) 새로운 노드를 추가하며, CAS 연산 성공 시 `break`문을 통해 반복문을 빠져나온다.

`casTabAt()` 메서드는 CAS 연산을 사용하여 입력 받은 버킷이 여전히 `null`일 때만 새로운 노드를 원자적으로 삽입한다.

```java
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
    return U.compareAndSetObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```

즉, `casTabAt()` 메서드는 다음을 수행한다.
- 해시 버킷의 값이 여전히 `null`인지 확인한다. (동시에 다른 스레드가 해당 위치에 노드를 저장했는지 체크)
- 값이 여전히 `null` 이면 새로운 노드를 원자적으로 저장하고 `true`를 반환한다.
- 그렇지 않다면 (다른 스레드가 해당 위치에 노드를 저장한 경우) `false`를 반환한다.

내부적으로 `casTabAt()`은 `compareAndSetObject()` 메서드를 호출하는데, 이 메서드는 네이티브 코드로 구현되어 있으며, JNI를 통해 자바 코드에서 네이티브 코드를 호출한다.

#### 조건부 노드 반환
```java
else if (onlyIfAbsent // check first node without acquiring lock
         && fh == hash
         && ((fk = f.key) == key || (fk != null && key.equals(fk)))
         && (fv = f.val) != null)
    return fv;
```
`onlyIfAbsent` 값이 `true`인 경우 (키가 없을 때만 삽입하는 케이스) 첫 번째 노드를 락 없이 확인하여, 키-값 쌍이 존재한다면 기존 값을 반환하고 메서드를 종료한다.

이 코드는 락을 획득하지 않고 미리 키의 존재 여부를 빠르게 검사함으로써, 불필요한 동기화(synchronized) 블록 진입을 건너뛰어 동시성 환경에서의 성능을 향상시키기 위한 최적화 로직이다.

#### 노드 추가: 버킷에 노드가 존재하는 경우 (synchronized)
![images](https://1drv.ms/i/c/9251ef56e0951664/IQSTLwgeSIusSK7BzTPrBsLXAaYcKOYaGLvhdfEWYO69opI)

`HashMap`과 비슷한 로직으로 구성된다. 동일한 키 값이 발견되면 해당 값을 새 데이터로 덮어쓰고, 동일한 키 값이 없으면 체인 맨 끝에 새 데이터를 추가하는 방식이다.

- (1) **버킷 레벨 락(Bucket-level Locking)** 전략<br/>
    `synchronized (f)` 블록은 해당 버킷의 첫 번째 노드 `f`를 **뮤텍스**로 사용하여 락을 건다. 이로 인해 특정 버킷에는 **오직 하나의 스레드만 접근**할 수 있도록 제어한다. 서로 다른 스레드가 같은 해시 버킷에 접근할 때만 락 경합이 발생한다.

    이러한 방식은 맵 전체를 잠그지 않고, 각 버킷에 대해 독립적으로 락을 건다. 따라서 한 스레드가 특정 버킷에 대해 작업 중일 때 다른 버킷에 영향을 주지 않는다. 따라서, 다른 스레드들은 **다른 버킷에 접근하여 작업을 진행**할 수 있다. 맵 전체에 단일 락을 거는 `Hashtable`이나 `Collection.synchronizedMap()` 방식과 달리, **락 경합을 최소화**하고 동시 처리량을 높일 수 있다.

- (2) `fh` 는  Hash Type을 나타내는 변수다. 해시 버킷 첫 번째 노드 `f`의 `hash` 필드 값을 가져온 것으로, 이 값은 해당 버킷이 어떤 구조로 데이터를 관리하는지 나타내는 플래그 역할을 한다. 이 값이 음수인지 양수인지에 따라 해당 버킷에 저장된 데이터 구조와 안전성 확보 방식이 결정된다.

    - `fh >= 0` : 데이터가 일반 노드(`Node<K,V>`) 또는 연결 리스트 구조로 저장되어 있음을 의미하며, 해시 충돌 발생 시 Separate Chaining 방식으로 데이터를 추가한다. 실제 키의 해시 값이 저장된다. `HashMap`의 `putAll()` 로직과 유사하다.

    - `fh < 0` : 해당 버킷의 노드가 특별한 제어 노드(Control Node)를 담고 있음을 의미한다. 이는 현재 트리 구조이거나 맵 전체가 재배치 작업 중인 특수 목적의 노드를 구분하기 위해 음수 해시 값을 예약해 두었다.

    | `fh` 값 | 구조적 의미 | 설명 |
    | ----- | ----- | ----- |
    | `-1` | `MOVED` | 재배치(Resizing) 또는 이동(Transferring) 작업 중임을 나타낸다.<br/>해당 버킷에 데이터를 추가하려는 스레드는 이동 작업을 돕도록 유도되어<br/>재배치 작업의 병렬성을 높인다. |
    | `-2` | `TREEBIN` | 해당 버킷이 트리 구조(TreeBin)으로 변환되었음을 나타낸다. |
    | `-3` | `RESERVED` | (주로 디버깅 및 내부 임시 용도) |


- (3) `onlyIfAbsent` 값이 `false`일 때만 값을 덮어 쓴다.

- (4) 노드가 트리 구조(`TreeBin`)로 되어 있다면, `putTreeVal()` 메서드를 호출하여 트리에 데이터를 추가한다.

- (5) 노드가 `ReservationNode` 이면 `IllegalStateException`을 예외를 발생시켜 재귀적 갱신을 방지한다.

    데이터를 이동하는 도중 다른 스레드가 이 버킷에 접근하면 발생할 수 있는 재귀적 갱신(Recursive Update) 문제를 방지하기 위해 `ReservationNode`로 표시하여 해당 인덱스가 이미 작업 중임을 표시한다.

    > **ReservationNode**란?<br/>
    > 실제 키-값 데이터를 저장하는 노드가 아니며, 특정 버킷이 재배치(Resizing/Transferring) 작업 중임을 표시하는 자리 표시자(Placeholder) 또는 제어 노드(Controller) 역할을 한다.
    >
    > - 재배치 표식
    >     - 내부적으로 크기를 조정하거나 데이터를 새로운 배열로 이동시키는 `transfer` 작업 수행할 때 사용된다.
    >     - 노드가 이동을 완료했음을 표시하기 위해 새로운 테이블의 해당 인덱스(버킷)에 `ReservationNode`를 배치한다. 다른 스레드가 이 버킷에 접근했을 때 해당 버킷의 데이터 이동이 완료되었음을 즉시 알 수 있게 한다.
    > - 구조적 위치
    >     - 일반적으로 버킷의 첫 번째 노드로 위치하며, 다른 제어 노드들과 마찬가지로 **음수 해시 값**을 가진다. (보통 `ConcurrrentHashMap` 코드에서 재배치 중임을 나타내는 주 해시 값은 `MOVED`를 나타내는 `-1`을 사용하며, `ReservationNode`는 `TreeBin`의 잠금을 확보할 때 등 더 구체적인 제어 목적으로 사용되는 경우가 많다.)
    >     
    {: .prompt-tip }

#### 병렬 리사이징
```java

else if ((fh = f.hash) == MOVED)
    tab = helpTransfer(tab, f);
```
`f.hash` 값이 `MOVED`인 경우, 테이블이 리사이징 중임을 의미한다. 이 때 `helpTransfer()` 메서드를 호출하여 현재 스레드가 리사이징 작업을 돕도록 한다. 병렬 리사이징을 통해 여러 스레드가 동시에 테이블 크기를 조정할 수 있게 한다.

#### 트리화 체크 및 기존 값 반환
```java
for (Node<K,V>[] tab = table;;) {
    ...
    else {
        ...
        synchronized (f) {
            ...
        }
        if (binCount != 0) {
            // 트리화 체크
            if (binCount >= TREEIFY_THRESHOLD)
                treeifyBin(tab, i);
            // 기존 값 반환
            if (oldVal != null)
                return oldVal;
            break;
        }
    }
```
**[트리화 체크]**<br/>
`synchronized` 블럭에서 노드를 추가하고 락이 해제된 직후, 버킷을 트리 구조로 변환할 지 체크한다. 버킷 내 노드 수가 `TREEIFY_THRESHOLD`(default 8)를 초과하면 연결 리스트를 트리 구조로 변환한다. 트리 구조는 리스트보다 빠른 검색 성능을 제공한다. <시간 복잡도 : `O(log(n))`>

**[기존 값 반환]**<br/>
`synchronized`에서 기존 키에 대한 값 업데이트가 발생한 경우, 업데이트 이전 값이 `oldVal`에 저장된다. `oldVal`이 `null` 이 아니라는 건 **기존 키-값 쌍을 덮어썼음**을 의미하므로 `put()` 메서드의 규약에 따라 이전 값을 반환하고 `putVal()` 메서드를 종료한다.

## Thinking. 왜 버킷이 비었을 때만 CAS 연산을 사용할까?
코드를 분석하며 문득 궁금증이 생겼다. 버킷이 사용 중일 때 락을 사용하는데, 왜 락 없이도 높은 성능으로 동시성 제어를 제공하는 CAS 연산을 굳이 **버킷이 비었을 때**만 사용할까? 

필자의 단순한 추측이지만, 아마도 CAS 연산이 매우 단순한 삽입 상황에 적합해서가 아닐까 싶다.

빈 버킷에 노드를 삽입하는 것은 이 버킷이 `null` 이면 새 노드를 넣어라는 단순한 논리를 CAS 연산으로 쉽게 구현할 수 있다.

생각해보면, 빈 버킷에 새 노드를 넣는 작업은 '이 버킷이 null이면, 새 노드를 넣는다'는 아주 단순한 논리이다. 이 정도의 작업은 CAS 연산 한 번으로 구현하기에 간단해 보인다..

그런데 버킷이 사용 중일 땐 이야기가 달라진다. 기존 버킷의 연결 리스트나 트리 구조를 따라 탐색, 비교하고 새로운 노드를 체인 끝에 추가하거나 기존 노드를 업데이트하는 과정이 매우 복잡하다.

만약 이러한 복잡한 작업에 CAS를 사용한다면 매번 전체 버킷의 상태를 표현하는 복잡한 CAS 조건을 정의하고, CAS 실패 시 재시도 로직이 너무 복잡해져 골치가 아플 것 같다.

이 복잡성 때문에 락을 한 번 잡고 작업하는 것이 CAS를 계속 시도하고 실패하며 재시도하는 비용보다 효율이 더 좋아 보인다.

그래서 `ConcurrentHashMap`은 가장 빈번하게 발생하는 단순한 케이스인 빈 버킷인 경우에는 CAS 연산으로 속도를 끌어올리고, 그렇지 않은 경우에는 `synchronized` 락 방식으로 안전하게 처리하는 전략을 채택했지 않을까 하는 생각이 들었다.

## References
- https://blog.hexabrain.net/403
- https://curiousjinan.tistory.com/entry/java-concurrent-hash-map-cas
- https://www.javamadesoeasy.com/2015/04/concurrenthashmap-in-java.html
- Gemini 2.5 Pro