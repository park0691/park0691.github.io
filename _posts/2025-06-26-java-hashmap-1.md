---
title: 'HashMap 해부 (1) : 구조와 데이터의 초기 저장 과정'
date: 2025-06-26 12:00 +0900
categories: Java
tags: hashmap,hash,collections
---
### HashMap, 그 익숙함 속의 깊이
Java 개발자라면 `HashMap`은 매우 익숙한 컬렉션이다. 키-값 쌍으로 데이터를 저장하고 빠르게 조회할 수 있어 자주 사용하곤 한다. 하지만 이 `HashMap`이 내부적으로 어떻게 동작하는지, 데이터가 어떤 과정을 거쳐 저장되고 관리되는지 깊이 들여다본 경험은 얼마나 될까? 사실 `HashMap`은 우리가 흔히 사용하는 단순한 겉모습과 달리, 뛰어난 성능을 유지하기 위해 복잡하고 정교한 내부 구조와 알고리즘을 품고 있다.

Java 해싱 및 해시맵 관련하여 공부한 지식적인 내용은

[https://park0691.github.io/TIL/java/collections-hashing.html](https://park0691.github.io/TIL/java/collections-hashing.html)

[https://park0691.github.io/TIL/java/collections-map.html](https://park0691.github.io/TIL/java/collections-map.html)

에 정리해두었다. 본 포스트에서는 이 기본 지식을 바탕으로 Java 8 `HashMap` 소스 코드를 직접 분석하며 해시맵이 내부적으로 어떻게 동작하고 데이터를 관리하는지 직접 확인하며, 그 숨겨진 원리들을 파헤쳐 보고자 한다.

#### 요약
이번 포스트에서는 다음 내용을 위주로 살펴본다.

- 해시맵의 성능을 좌우하는 [주요 상수](#주요-상수)들과 [핵심 멤버 변수](#멤버-변수)
- [보조 해시 함수 : 해시 코드 연산 시 왜 XOR 연산을 할까?](#보조-해시-함수)
- [생성자 : 해시테이블의 지연 초기화](#생성자)
- [`put()`의 숨겨진 동작 : HashMap이 생성된 후 데이터를 처음 저장할 때 내부적으로 어떤 과정이 일어날까?](#데이터-저장)

### 주요 상수
1. DEFAULT_INITIAL_CAPACITY
```java
/**
 * The default initial capacity - MUST be a power of two.
 */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
```
- 해시맵의 디폴트 버킷 사이즈다.
- 해시 테이블의 배열 길이 초기값으로 사용된다.
- 비트 마스크 연산 최적화를 위해 반드시 2의 제곱 수이어야 한다.

2. MAXIMUM_CAPACITY
```java
/**
 * The maximum capacity, used if a higher value is implicitly specified
 * by either of the constructors with arguments.
 * MUST be a power of two <= 1<<30.
 */
static final int MAXIMUM_CAPACITY = 1 << 30;
```
- 해시 테이블의 최대 크기로 배열의 길이는 이 값을 넘을 수 없다.
- 2^30 = 1,073,741,824

3. DEFAULT_LOAD_FACTOR
```java
/**
 * The load factor used when none specified in constructor.
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```
- 해시맵의 기본 Load Factor(부하율)를 의미한다.
- Load Factor를 초과하면 리사이징 발생한다.

4. TREEIFY_THRESHOLD
```java
/**
 * The bin count threshold for using a tree rather than list for a
 * bin.  Bins are converted to trees when adding an element to a
 * bin with at least this many nodes. The value must be greater
 * than 2 and should be at least 8 to mesh with assumptions in
 * tree removal about conversion back to plain bins upon
 * shrinkage.
 */
static final int TREEIFY_THRESHOLD = 8;
```

- 해시 버킷 하나에 `TREEIFY_THRESHOLD` 개수 이상의 요소가 충돌하면 트리 구조(Red-Black Tree)로 변환한다.
    - 특정 해시 버킷에 데이터가 많이 충돌하여 연결 리스트의 길이가 8 이상이 되면 해시맵은 해당 버킷의 성능 저하(최악인 경우 O(N))를 방지하기 위해 연결 리스트를 이진 탐색 트리(Red-Black Tree)로 변환하여 탐색 시간을 O(logN)으로 개선한다.


5. UNTREEIFY_THRESHOLD
```java
/**
 * The bin count threshold for untreeifying a (split) bin during a
 * resize operation. Should be less than TREEIFY_THRESHOLD, and at
 * most 6 to mesh with shrinkage detection under removal.
 */
static final int UNTREEIFY_THRESHOLD = 6;
```
- 트리 구조였던 해시 버킷의 요소가 6개 미만이 되면 다시 리스트 구조로 변환한다.


6. MIN_TREEIFY_CAPACITY
```java
/**
 * The smallest table capacity for which bins may be treeified.
 * (Otherwise the table is resized if too many nodes in a bin.)
 * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
 * between resizing and treeification thresholds.
 */
static final int MIN_TREEIFY_CAPACITY = 64;
```
- 해시 테이블의 전체 크기가 64 미만이면 트리로 변환하지 않고 리사이징을 우선한다.
    - 이는 테이블 크기가 작을 때 특정 버킷만 트리로 만드는 것보다 전체 테이블을 확장하여 해시 충돌 가능성을 줄이는 것이 더 효율적이라 판단하기 떄문이다. 트리는 일반 노드보다 메모리 사용량이 많으므로, 작은 맵에서 트리화는 성능이 오히려 떨어질 수 있다.

> **해시 버킷의 연결 리스트가 트리 구조로 변환되는 조건**<br/>
> 다음 두 가지 조건을 다 충족하여야 한다. <br/>
> 1. 버킷 내 노드 개수 임계값 초과 : 해당 버킷에 저장된 엔트리(Node) 개수가 `TREEFIFY_THRESHOLD` 상수값인 8개 이상이 될 때
> 2. 전체 테이블 용량 임계값 초과 : 전체 테이블 용량(capacity)이 `MIN_TREEIFY_CAPACITY` 상수값인 64 이상이 될 때
{: .prompt-warning }

### 멤버 변수
1. table
   
    ![images](https://1drv.ms/i/c/9251ef56e0951664/IQTQQEFklKKmQ4715b6dywUwAblZpIqjiDbwO8g93ExmLms)
    _출처 : https://www.javaquery.com/2019/11/how-hashmap-works-internally-in-java.html_
    
    ```java
    /**
     * The table, initialized on first use, and resized as
     * necessary. When allocated, length is always a power of two.
     * (We also tolerate length zero in some operations to allow
     * bootstrapping mechanics that are currently not needed.)
     */
    transient Node<K,V>[] table;
    ```
    - 해시 테이블 (또는 해시 버킷)
    - 배열 구조로 저장되며 `Node<K, V>` 객체(또는 트리 노드)의 연결 리스트 또는 트리를 가리킨다.

    **[Node의 구조]**
    ```java
    /**
     * Basic hash bin node, used for most entries.  (See below for
     * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
    
        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
    
        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }
    
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }
    
        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }
    
        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
    ```
    - `key, value` 쌍뿐만 아니라 키의 해시값 `hash`, 다음 노드를 가리키는 `next` 변수가 들어 있다.
    - `java.util` 패키지의 `Map.Entry<K,V>` 인터페이스를 구현하였다.

    > **Map.Entry<K,V> 인터페이스**<br/>
    > Map 인터페이스의 Inner Interface로 Map에 저장되는 개별 요소인 key-value 쌍을 한 번에 다루기 위해 내부적으로 Entry 인터페이스를 정의하였다.<br/>
    > Map 인터페이스를 구현하는 클래스에서는 Map.Entry 인터페이스도 함께 구현해야 한다.
    > 
    > **[주요 메소드]**
    > 
    > | 메소드 | 설명 |
    > | ------- | ------- |
    > | boolean equals(Object o) | 동일한 Entry인지 비교한다. |
    > | K getKey() | Entry의 key 객체를 반환한다. |
    > | V getValue() | Entry의 value 객체를 반환한다. |
    > | int hashCode() | Entry의 해시코드를 반환한다. |
    > | V setValue(V value) | Entry의 value 객체를 지정된 객체로 변경한다. |
    > 
    {: .prompt-tip }


2. entrySet
```java
/**
 * Holds cached entrySet(). Note that AbstractMap fields are used
 * for keySet() and values().
 */
transient Set<Map.Entry<K,V>> entrySet;
```
- `entrySet()` 메서드 호출시 리턴된다.
- key-value를 한 번에 다뤄야 할 때 `keySet()`와 `get()`를 사용하여 따로따로 처리하는 경우 효율적이지 않다. 이 때, `entrySet()` 메서드를 사용하면 key-value를 함께 처리할 수 있어 더 효율적이다.


3. size
```java
/**
 * The number of key-value mappings contained in this map.
 */
transient int size;
```
- 해시맵에 저장된 엔트리(key-value 쌍)의 개수


4. modCount
```java
/**
 * The number of times this HashMap has been structurally modified
 * Structural modifications are those that change the number of mappings in
 * the HashMap or otherwise modify its internal structure (e.g.,
 * rehash).  This field is used to make iterators on Collection-views of
 * the HashMap fail-fast.  (See ConcurrentModificationException).
 */
transient int modCount;
```

    - 해시맵의 구조 변경 횟수 (e.g. 리해싱)
    - `fail-fast` 동작을 위한 값으로, `Iterator`에서 `CooncurrentModificationException` 발생에 사용된다.



5. threshold
```java
/**
 * The next size value at which to resize (capacity * load factor).
 *
 * @serial
 */
// (The javadoc description is true upon serialization.
// Additionally, if the table array has not been allocated, this
// field holds the initial array capacity, or zero signifying
// DEFAULT_INITIAL_CAPACITY.)
int threshold;
```

    - `capacity` * `load factor` 값
        - size가 이 값을 초과하면 내부적으로 해시맵을 리사이징한다. (임계값)
    - default일 때 capacity = 16, load factor = 0.75이므로 threshold = 12다.
        - HashMap에 12개 엔트리가 들어오면 해시맵의 capacity를 2배 늘리고 리사이징한다.

6. loadFactor
```java
/**
 * The load factor for the hash table.
 *
 * @serial
 */
final float loadFactor;
```

    - 적재율이라고도 하며, 버킷의 로드 비율을 나타내는 변수다.
    - 해시 테이블이 얼마나 가득 차 있을 때 리사이징을 할지 결정하는 역할을 한다.
    - 자세한 내용은 다음 참조
        - [https://park0691.github.io/TIL/java/collections-map.html#capacity%E1%84%8B%E1%85%AA-load-factor](https://park0691.github.io/TIL/java/collections-map.html#capacity%E1%84%8B%E1%85%AA-load-factor)

### 생성자
HashMap의 생성자는 다음 세 가지가 있다. 내부를 보니 생성자 호출 시점에 `loadFactor, threshold` 멤버 변수만 셋팅하며, 해시 테이블(버킷 배열, `table`)이 즉시 초기화되지 않는 점이 눈에 띈다.

```java
// 디폴트 생성자
public HashMap();

// 초기 용량만 설정
public HashMap(int initialCapacity);

// 초기 용량, 로드 팩터 설정
public HashMap(int initialCapacity, float loadFactor);
```

1. 디폴트 생성자
   
    ```java
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
    ```
    - 객체가 생성될 때 버킷 배열이 즉시 생성되지 않는다. 단순히 `loadFactor` 변수에 상수인 `DEFAULT_LOAD_FACTOR`를 할당하고만 있다.

2. 생성자 [1]
   
    ```java
    /**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and load factor.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
    ```
    - 파라미터로 받은 initialCapacity, loadFactor로 해시 테이블을 생성한다.
    - 11L | capacity 값 유효성 체크 (0 미만)
    - 14L | 최대 CAPACITY 초과한 capacity 들어온 경우 MAXIMUM_CAPACITY로 초기화
    - 16L | load factor 유효성 체크 (0 이하, NaN)
    - 19-20L | loadFactor, threshold 멤버 변수 셋팅
    
    ```java
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
    ```
    - `tableSizeFor()` 메소드는 threshold를 계산할 때 입력된 `cap` 값보다 크거나 같은 2의 제곱수를 리턴한다. capacity, threshold를 2의 제곱 수로 맞추기 위해 사용되는 것 같다.
        - 1 ~ 2 : returns 2
        - 3 ~ 4 : returns 4
        - 5 ~ 8 : returns 8
        - 9 ~ 16 : returns 16
        - 17 ~ 32 : returns 32

3. 생성자 [2]
   
    ```java
    /**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and the default load factor (0.75).
     *
     * @param  initialCapacity the initial capacity.
     * @throws IllegalArgumentException if the initial capacity is negative.
     */
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
    ```
    - 로드 팩터 없이 capacity 만 파라미터로 생성하는 경우 디폴트 로드 팩터값을 파라미터로 담아 바로 위에서 확인한 생성자 [1]을 호출한다.


### 보조 해시 함수
해시 테이블 인덱스를 계산하는 데 사용되는 최종 해시 값을 생성한다. 키 객체의 `hashCode()`를 그대로 사용하지 않고, `hashCode()` 결과를 가져와서 상위 비트와 하위 비트를 XOR 하는 것을 확인할 수 있다.

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

`hash = hash ^ (hash >>> 16)`

1. key의 해시 코드를 구한다.
2. 해시 코드의 부호 무시하고 16번 오른쪽 쉬프트
3. 두 값으로 XOR 연산 수행

> **해시 연산에 오른쪽 쉬프트, XOR 연산이 왜 포함될까?**<br/>
> 해시 테이블의 크기는 보통 2의 제곱 수로 표현됨을 위에서 확인하였다.
> 최종 인덱스를 계산하기 위해 `hash & (capacity - 1)` [`hash % capacity`와 같음] 연산한다.
> 
> 해시 테이블의 크기(capacity)가 16이라 가정하자. 인덱스를 계산할 때, `hash & 15` (`hash & 0b...00001111`) 연산을 수행한다. 즉, 해시 코드의 하위 비트만 사용하고 상위 비트는 무시한다.
> 
> 두 키의 hashCode() 반환 값이 우연히 하위 비트에서는 매우 비슷하고, 상위 비트만 차이 나는 경우, `hash & (capacity - 1)` 연산 시 많은 해시 충돌이 발생할 수 있다.
>
> **e.g.** hashCode() 값이 0xABCDEF01, 0xABCDEF02인 두 객체가 있다면, 하위 2비트 0x1, 0x2만으로 인덱스 계산하게 되어 충돌이 발생하지 않는다.<br/>
>   ![images](https://1drv.ms/i/c/9251ef56e0951664/IQR3LMpgvsG1SIpwroDNmVI8AVXssRDzz7TMdxrwCa9NsDc)
>   
> 만약 hashCode() 값이 0x12345678, 0x9ABCDEF8처럼 상위 비트에서만 큰 차이를 보이고 하위 비트가 동일하다면, hash & 15 연산 시 동일한 값이 나와 해시 충돌이 발생한다.
>   ![images](https://1drv.ms/i/c/9251ef56e0951664/IQQVxzYKHMdZTIayQuViOer3AdcFjAPCGS1LuZSi9vWOugk)
>   
> `hash()` 메서드의 `hash ^ (hash >>> 16)` 연산은 해시 코드의 상위 16비트와 하위 16비트를 섞는다. 이렇게 섞인 해시 값을 사용하면 해시 코드 상위 비트의 정보도 최종 인덱스 계산에 영향을 미쳐서, 하위 비트의 편향성으로 인한 해시 충돌 가능성을 줄일 수 있다. 따라서 해시 값을 해시 테이블 전체에 더 고르게 분산시키는 데 도움이 된다. 이는 해시맵의 성능을 유지하는 데 중요한 역할을 한다.
>   ![images](https://1drv.ms/i/c/9251ef56e0951664/IQT1dSfvXD7oSKADV73W7uw1AVnWkugBib1z7sx5JZVPjkA)
>   
> 정리하자면, `HashMap.hash()` 메서드는 객체의 hashCode() 결과를 가져와서 상위 비트와 하위 비트를 XOR 연산하여 해시 코드의 모든 비트가 해시 테이블 인덱스 계산에 최대한 영향을 미치도록 함으로써, 해시 충돌을 줄이고 HashMap의 성능을 최적화하는 역할을 한다. → 해시 분포를 더 균일하게!
>
{: .prompt-tip }


### 데이터 저장
HashMap에 실제 데이터를 추가할 때 put() 메서드가 호출될 때 생성된다.

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

`put()` 은 내부적으로 `hash()` 메서드를 호출한 후 `putVal()`을 호출한다.

**[putVal() 메서드]**

```java
/**
 * Implements Map.put and related methods.
 *
 * @param hash hash for key
 * @param key the key
 * @param value the value to put
 * @param onlyIfAbsent if true, don't change existing value
 * @param evict if false, the table is in creation mode.
 * @return previous value, or null if none
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```
- 입력 파라미터
    - `hash`, `key`, `value`는 이해가 쉬우므로 생략
    - `onlyIfAbsent` : `false`로 설정하면 해당 `key`가 해시맵에 이미 존재하면 새로운 값으로 덮어쓰지만, `true`로 설정하면 해당 `key`가 해시맵에 이미 존재하는 경우 덮어쓰지 않는다.
    - `evict` : `HashMap` 자체에서는 이 파라미터가 직접적인 동작을 유발하지 않는다. `afterNodeInsertion()`와 같은 메소드를 호출하는 데 사용되며, `LinkedHashMap`에서 캐시 제거 정책(LRU 등)을 구현하는 데 오버라이드될 수 있다.
- 50L | `afterNodeInsertion()` 코드는 주로 `LinkedHashMap`에서 구현하는 부분으로 `HashMap`에서는 직접적인 동작을 유발하지 않는다.

#### 데이터를 처음 저장할 때
앞서 해시맵 생성 시점에 해시 테이블이 초기화되지 않는 것을 확인했다. 해시맵에 데이터를 처음 저장하는 시점에 해시테이블이 초기화된다. 위 코드 중 아래의 두 `if` 문이 해시맵에 데이터를 처음 저장할 때 수행되는 핵심 로직이다.
![images](https://1drv.ms/i/c/9251ef56e0951664/IQT4PkAe6r8LTJGqEQQSHwrmAdbt1xxMtrwDRLxCP-9pWvc)
_HashMap 생성 후 데이터를 처음 저장(put() 호출)할 때 putVal()에서 수행되는 코드_

**[1]**
```java
if ((tab = table) == null || (n = tab.length) == 0)
    n = (tab = resize()).length;
```

- `(tab = table) == null` : table은 `null` (초기화되지 않음) 이므로 `true`
- `n`에 `resize()`를 호출하여 생성된 해시 테이블의 길이를 받는다.

**[1 - resize() 메소드]**
- 해시 테이블의 크기를 동적으로 조절하여 성능을 유지하는 데 핵심적인 역할을 한다.
- 주로 다음 두 가지 경우에 호출된다.
    - 초기화 (Lazy Initialization) : 해시맵을 처음 생성하고 첫 번째 요소를 추가할 때 내부 배열(`table`)이 아직 초기화되지 않았다면 호출되어 기본 초기 용량(16)으로 배열을 생성한다.
    - 용량 초과 (Load Factor Threshold) : 해시맵에 저장된 엔트리(key-value 쌍) 개수가 `threshold` 값을 초과할 때 호출된다.

```java
/**
 * Initializes or doubles table size.  If null, allocates in
 * accord with initial capacity target held in field threshold.
 * Otherwise, because we are using power-of-two expansion, the
 * elements from each bin must either stay at same index, or move
 * with a power of two offset in the new table.
 *
 * @return the table
 */
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

데이터를 처음 저장하면 `resize()` 코드에서 아래 코드가 수행된다.
![images](https://1drv.ms/i/c/9251ef56e0951664/IQRCH0H5KHueSqa7zqzut-kVAZo5Vfe2zxz60-yig3q82qY)
_HashMap 생성 후 데이터를 처음 저장(put() 호출)할 때 resize()에서 수행되는 코드_

- 15L | `if (oldCap > 0)` : `oldCap = 0` 이므로  `false`
- 24L | `else if (oldThr > 0)` : `oldThr = 0` 이므로 `false`
- 26 ~ 29L | `newCap`에  DEFAULT_INITIAL_CAPACITY(16)을 할당하고 `newThr`에는 로드팩터와 Capacity를 곱한 값(12)을 할당한다.
- 37L | `newCap`(16) 용량의 `Node<K,V>` 타입 배열(해시 테이블)을 생성 후
- 39L | `if (oldTab != null)` : `oldTab = null` 이므로 `false`
- 81L | 생성한 해시 테이블을 리턴한다.

그 결과, 다음과 같은 구조의 해시 테이블이 생성되었다.

![images](https://1drv.ms/i/c/9251ef56e0951664/IQTPQ7URxqhgTaKU8xKBcaRVAbmHaKA111VRGd6ZUfM62Ak)
_resize() 결과 생성된 해시 테이블(table) 구조_

**[2]**
```java
if ((p = tab[i = (n - 1) & hash]) == null)
    tab[i] = newNode(hash, key, value, null);
```

- `n` : 해시 테이블의 길이를 받는다.
- `hash` :  HashMap의 보조 해시 함수인 `hash(key)` 메소드로 반환된 값을 받는다.
- `i = (n - 1) & hash` : `i = hash % n`과 동일하다. (해시 테이블 인덱스로 맵핑)
- `(p = tab[i]) == null` : 데이터가 하나도 저장되지 않아 해시 테이블은 비어 있으므로 `true`
- `tab[i] = newNode(hash, key, value, null)` : 해시 테이블에 엔트리를 생성하여 집어넣는다.

**[2 - newNode() 메소드]**
- 해시 테이블 엔트리를 생성하는 `newNode()` 메소드는 다음과 같다. 단순히 생성자를 호출하여 생성된 객체를 반환한다.

```java
// Create a regular (non-tree) node
Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
    return new Node<>(hash, key, value, next);
}
```

#### 실제 동작 살펴보기

예를 들어 다음과 같은 간단한 구조의 해시맵에 데이터를 넣었다 가정하자.

![images](https://1drv.ms/i/c/9251ef56e0951664/IQSGmbx73JniSbRjk67wELEXATp--5tLqg-d8mD3R_pEgsQ)

해시맵 생성 직후(1L 수행 후) 해시맵 주요 멤버를 보면,

![images](https://1drv.ms/i/c/9251ef56e0951664/IQSg-Q3nf-Q3T7mUO8bPjMYAAXox-nbtZAvs2HyiD5KW784)

- `table`은 초기화되지 않아 `null`
- `loadFactor`은 디폴트 생성자에 의해 초기화되어 `0.75`
- `threshold`는 초기화되지 않아 `0`

임을 확인할 수 있다.

`put()` 메소드 호출 시 내부 동작을 보자

![images](https://1drv.ms/i/c/9251ef56e0951664/IQQI7A70MdlcQqKs3YXrBoTuAe5y8S2VMemhuHMYX2zecAM)

- `put()`은 보조 해시 함수인 `hash(key)`에서 리턴 받은 값 `2301537`을 담아 `putVal()` 메소드를 호출하는 것을 확인할 수 있다.
- `putVal()`은 해시 테이블인 `table`이 초기화되지 않아 `resize()` 메소드를 호출한다.

![images](https://1drv.ms/i/c/9251ef56e0951664/IQRuLPCV1v8jQ4ZTKrg4oZt6AeSJVjKD0JhskYOnFnq2VwU)

- `resize()` 메소드 리턴 시점에서 해시맵 내부 멤버 변수를 보면
    - 해시 테이블 `table` 이 길이 `16`의 `Node` 타입 배열로 초기화된 것을 확인할 수 있고,
    - `threshold` 값도 `12`로 초기화된 것을 확인할 수 있다.

`resize()` 호출 후의 `putVal()` 동작을 보면 (`if ((p = tab[i = (n - 1) & hash]) == null)` 코드 라인부터)

![images](https://1drv.ms/i/c/9251ef56e0951664/IQQZKDObEumDTo8aHai5xKrNAa4195dxwVPL2BayJ8Nu0sI)


- `i = (n - 1) & hash`
    - `i = 15 & 2301537` 
    - `i = 1` 이므로 해시 테이블 인덱스 `1` 위치에 데이터가 저장된다.
- `newNode()` 메소드로 새 `Node` 엔트리를 생성하여 `table[1]`에 저장한다.
- `putVal()` 메소드 리턴 시점에서 해시 테이블 내부 멤버 변수를 보면
    - 실제로 `table[1]` 에 저장한 데이터가 들어가있는 것을 확인할 수 있고,
    - `size`가 `1` 증가된 것을 확인할 수 있다.

내부 테이블을 더 정확히 살펴보기 위해 IntelliJ 설정을 바꿔서 내부 Node 멤버 변수가 보이도록 셋팅한 뒤 확인하면 더 정확히 확인할 수 있다. (객체의 레퍼런스는 일부 바뀌어 보임)

![images](https://1drv.ms/i/c/9251ef56e0951664/IQQZU_xZ6KDDSLZ2Tq0QqBGnAWWuWin3yKFLowwC3PxkJII)

위 코드 수행 후 해시 테이블 구조를 그러보면

![images](https://1drv.ms/i/c/9251ef56e0951664/IQSfkNgTC-qPRKresbyyPOdFAdJZGu0c6AGTqxnfbJRx79k)

## References
- gpt4o
- Gemini 2.5 Flash
- https://docs.oracle.com/javase/8/docs/api/java/util/Map.Entry.html
- https://lordofkangs.tistory.com/78
- https://mangkyu.tistory.com/430