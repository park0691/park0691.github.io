---
title: 'HashMap 해부 (2) : 해시 충돌'
date: 2025-07-03 12:00 +0900
categories: Java
tags: hashmap,hash,collections
---

#### 요약
이번 포스트에서는 다음 내용을 위주로 살펴본다.

- 해시 충돌을 의도적으로 발생시켜, 해시맵이 체이닝 방식으로 동일한 버킷에 어떻게 여러 엔트리를 저장할까?
- 이미 존재하는 키에 새로운 값을 저장할 때, 해시맵이 기존 값을 어떻게 덮어쓰는지 알아본다.
- `hashCode()`와 `equals()`가 어떻게 활용되어 기존 키와 새 키를 비교할까?
- 트리화, 해시 버킷에 8개 이상의 엔트리가 저장될 때 트리 구조로 변환하는 코드를 간략하게 살펴본다.

### 데이터 저장
앞의 포스트에서 HashMap에 실제 데이터를 추가할 때 호출된 `put()` 메서드에서는 내부적으로 `putAll()` 메서드를 호출함을 확인하였다.

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

또한 `put()` 은 내부적으로 해시 보조 함수인 `hash()` 메서드를 호출하여 키 값을 최대한 고르게 한 후 `putVal()`을 호출한다. `putVal()` 메소드는 다음과 같다.

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

#### 다른 키가 동일한 해시 버킷에 매핑될 때
둘 이상의 다른 `key`(`equals()` 메서드 결과가 `false`)가 동일하거나 유사한 해시 값을 반환하여 HashMap 내부 해시 함수를 거쳐 같은 버킷에 할당되는 해시 충돌 상황을 인위적으로 유도하고자 한다.

이를 위해서는 `hashCode()` 메서드가 동일한 값을 반환하면서 `equal()` 메서드는 `false`를 반환하는 커스텀 키 클래스를 만들어서 두 개의 다른 객체가 `HashMap`의 동일한 해시 버킷에 저장되도록 강제할 수 있다.

커스텀 키 클래스는 다음과 같이 작성했다.

```java
class CollisionKey {
    private final int fixedHash; // 항상 동일한 해시 코드를 반환하도록 고정
    private final String value;  // equals() 비교에 사용될 실제 값

    public CollisionKey(int fixedHash, String value) {
        this.fixedHash = fixedHash;
        this.value = value;
    }

    @Override
    public int hashCode() {
        // 항상 동일한 해시 코드를 반환하여 충돌을 유도한다.
        return fixedHash;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        CollisionKey that = (CollisionKey) o;
        // value 필드가 다르면 다른 객체로 간주
        return value.equals(that.value);
    }

    @Override
    public String toString() {
        return "CollisionKey(fixedHash=" + fixedHash + ", value='" + value + "')";
    }
}
```

**[예시 코드]**

```java
int commonHash = 1; // 모든 CollisionKey 인스턴스가 이 해시 코드를 가진다.

CollisionKey key1 = new CollisionKey(commonHash, "FirstValue");
CollisionKey key2 = new CollisionKey(commonHash, "SecondValue"); // key1과 해시 충돌
CollisionKey key3 = new CollisionKey(commonHash, "ThirdValue");  // key1, key2와 해시 충돌
CollisionKey key4 = new CollisionKey(commonHash, "FourthValue"); // 계속해서 해시 충돌

HashMap<CollisionKey, String> map = new HashMap<>();

// map1에 키-값 쌍 추가 (일부러 충돌 유도)
map.put(key1, "Value for Key1");
map.put(key2, "Value for Key2");
```

두 번째 데이터 `key2` 삽입(12L) 직전 해시 테이블 상태는 다음과 같다.
![images](https://1drv.ms/i/c/9251ef56e0951664/IQSa22SaypfgR7ob9k1n_ojPAc0SEOfAQFRcgVavj-Yyx_4)

1번 인덱스 위치의 해시 버킷에 엔트리가 하나 들어 있다.

두 번째 데이터 저장을 위한 `put()` 메소드 호출 시 `putVal()` 동작을 살펴보자.

![images](https://1drv.ms/i/c/9251ef56e0951664/IQQqRMFv5WJpSocHKKQI_zjbAZmvaX5OmHynHIt7cbhSn2U)

두 번째 데이터 `key2` 삽입 시 호출된 `putVal()` 메소드의 파라미터 `hash` 값은 `1`로  첫 번째 데이터 호출시 삽입한 키 `key1`의 `hash` 값과 동일하다.

- 629L | 해시 테이블 이미 생성되어 있으므로 `false`
- 631L | `if ((p = tab[i = (n - 1) & hash]) == null)`
    - `i = 15 & 1 = 1` 이므로 해당 해시 버킷에는 엔트리가 하나 이미 있으므로 `false`
- 635L | `p.hash == hash` : `true`
    - `hash` 값은 `1`로 동일
- 636L | `(k = p.key) == key` : `false`
    - `p.key` : 해시 버킷에 저장된 엔트리(첫 번째 저장된 키)
    - 첫 번째 저장된 키와 두 번째 저장할 키 `key`는 같은 객체가 아니므로 `false`
- 636L | `key != null && key.equals(k)` : `false`
    - `key != null`은 `null`이 아니므로 `true`
    - `key`와 `k`는 동등하지 않아 `equals()` 반환 값은 `false` (객체 내 `value` 값 달라서)
- 638L | `<Node K, V> p` 객체는 `TreeNode` 아니므로 `false`

> ★★★ 635 - 636L의 `if`문 조건을 통해서 해시맵에서 키 값 비교 로직을 확인할 수 있다. 왜 `key` 클래스의 `hashCode(), equals()` 메서드를 제대로 오버라이딩해야 하는지 확인할 수 있는 부분이다. `Hash` 기반 컬렉션은 우선적으로 해시코드로 같은 객체인지 판단하고, 같은 경우 다시 한번 `equals()` 메서드로 같은 객체인지 비교한다.
{: .prompt-tip }


두 `key` 객체는 `hashCode()`는 동일하지만 `equals()` 비교 결과 다른 값이므로 다른 객체로 판단한다.

![images](https://1drv.ms/i/c/9251ef56e0951664/IQSyRdwO33tdSZ0heS7ytJ1fAd-1LhnCaFH7hnntgU-JOVk)

위의 코드문(641L)이 수행된다면, 해당 해시 인덱스 위치의 버킷에 엔트리가 있는 상태이고 해당 엔트리의 키값이 저장하려는 엔트리의 키와 다르다 판단한 상황이다. 이 경우 해당 버킷의 체이닝(연결 리스트)으로 연결된 나머지 엔트리도 순차적으로 검사하여 같은 키가 없는지 판단해야 한다.

- 641L : 해당 해시 버킷 리스트를 탐색할 때 빈 카운트를 증가시키며 탐색한다.
- 642L : 마지막 노드까지 도달한 경우, 저장하려는 키와 일치하는 엔트리가 없다는 의미다.
- 643L : 따라서 엔트리를 새로 생성하여 체이닝 방식으로 연결 리스트 맨 끝에 엔트리를 저장한다.
- 644-645L : 만약 해당 해시 버킷에 8개 이상의 엔트리가 저장된 경우 해시맵을 트리화한다.

위 코드를 통해 체이닝(연결 리스트) 방식으로 저장하는 동작을 확인하였고, 해시 버킷 내 8개 이상의 엔트리가 저장되면 트리화하는 것도 확인할 수 있다.

`putAll()` 메소드 리턴 시점에서 해시맵 내부 멤버 변수를 보면

![images](https://1drv.ms/i/c/9251ef56e0951664/IQQ3GzOlNpehQ6RYW_-lMotlAfduS9cotalaM1a_9ospKp4)

- 해시 테이블의 1번째 인덱스 해시 버킷에 첫 번째 엔트리가 잘 저장되어 있고
- 첫 번째 엔트리 다음 엔트리 `this.table[1].next`로 방금 저장한 두 번째 데이터가 잘 저장된 것을 확인할 수 있다.

위 코드 수행 후 해시 테이블 구조를 그려보면
![images](https://1drv.ms/i/c/9251ef56e0951664/IQSjTJ5SrDiRTL0msNuqAFJ7AdWusr8Q72wVnBrSS8mTZlM)

마찬가지 방식으로 2개의 엔트리를 더 저장하자

```java
map.put(key1, "Value for Key1");
map.put(key2, "Value for Key2");
map.put(key3, "Value for Key3");
map.put(key4, "Value for Key4");
```

위 코드가 수행된 뒤 내부 테이블을 살펴보면,

![images](https://1drv.ms/i/c/9251ef56e0951664/IQSs_-HGuqZeT5Jj1s2MRk_tAW8Z9x8ICGQrqE5ddeC8L8Q)

내부 테이블 구조를 그려보면 다음과 같다.

![images](https://1drv.ms/i/c/9251ef56e0951664/IQSSA5ABfOr5TIKgrwsL1ZOFATAGxC7K0GHRgYRo5WOZKxY)

#### 동일한 키 엔트리가 맵에 존재할 때
이번에는 바로 위의 상황처럼 엔트리가 4개 저장된 뒤, 3번째 키와 똑같은 키로 새로운 값을 덮어쓰는 상황을 가정하자. 이를 위해 `equals()`가 `true`는 반환하도록 해야 한다.

```java
map.put(key1, "Value for Key1");
map.put(key2, "Value for Key2");
map.put(key3, "Value for Key3");
map.put(key4, "Value for Key4");

CollisionKey sameKey = new CollisionKey(commonHash, "ThirdValue");
map.put(sameKey, "Overwritten Value");
```

`sameKey`는 `key3`와 같다. (즉 `hashCode()`가 같고, `equals()` 리턴값이 `true`)

7L 코드가 수행될 때 `putAll()` 메서드 내에서는 다음 동작이 수행된다.

![images](https://1drv.ms/i/c/9251ef56e0951664/IQRCicwhR8vPSZqxQvDtpu_lAdNXZ5tFsTVGKwdqsGzlnvE)

- 641L | for 문이 수행되어 해시 빈을 순차 탐색한다.
- 648L | 같은 `key` 객체가 있는지 체크한다.
    - 3번째 객체의 `hash`가 같고, `equals()` 연산 결과 `true`가 나와서 `for`문을 빠져나온다.

![images](https://1drv.ms/i/c/9251ef56e0951664/IQQdqmPX5r9QT4wGN_JWZfSUARI3kArrWgQvjkN3jgdXVs0)

이후 654L 조건문이 수행된다.

- 654L |  `!onlyIfAbsent : true`
- 657L | 해당 엔트리의 `value`를 덮어 쓴다.

해당 작업 수행 후 결과 데이터를 보자.

![images](https://1drv.ms/i/c/9251ef56e0951664/IQQ2H6HM74ZsTLdTPGf4wPMOAdFsCxdS1u7PrDRp765uAPs)

1번째 해시 버킷의 3번째 엔트리의 `value`가 업데이트되었음을 확인할 수 있다.

![images](https://1drv.ms/i/c/9251ef56e0951664/IQRMULTIE0BeQIgTZHDYMOLLAdvJUot2F0wLfBEX2cwv-1k)


## References
- gpt4o
- Gemini 2.5 Flash
- https://docs.oracle.com/javase/8/docs/api/java/util/Map.Entry.html
- https://lordofkangs.tistory.com/78
- https://mangkyu.tistory.com/430