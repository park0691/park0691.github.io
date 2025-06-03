---
title: 불변 객체를 쉽게 다루는 클래스 타입 Record
date: 2025-05-30 17:00 +0900
categories: Java
tags: record, immutable
---

## Record
### 필요성
우리는 일반적을 DB 쿼리 결과, 서비스의 정보 등 데이터를 보관하기 위해 클래스를 작성하는데, 이러한 경우 데이터는 불변인(`immutable`) 경우가 많다.

보통 불변 클래스는 다음을 포함한다.

1. 각 데이터에 대한 *private*, *final* 필드
2. 각 필드에 대한 *getter*
3. 각 필드에 해당하는 인수가 있는 *public constructor*
4. 모든 필드가 일치할 때 동일한 클래스의 객체에 대해 *true*를 반환하는 *equals()* 메서드 
5. 모든 필드가 일치할 때 동일한 값을 반환하는 *hashCode()* 메서드
6. 클래스 이름과 각 필드 이름 및 해당 값을 포함하는 *toString()* 메서드

다음과 같은 불변 클래스 `Person`이 있다고 가정하자.

```java
public class Person {

    private final String name;
    private final String address;

    public Person(String name, String address) {
        this.name = name;
        this.address = address;
    }

    public String name() { return name; }
    public String address() { return address; }

    @Override
    public int hashCode() {
        return Objects.hash(name, address);
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) {
            return true;
        } else if (!(obj instanceof Person)) {
            return false;
        } else {
            Person other = (Person) obj;
            return Objects.equals(name, other.name)
              && Objects.equals(address, other.address);
        }
    }

    @Override
    public String toString() {
        return "Person [name=" + name + ", address=" + address + "]";
    }
}
```

불변 클래스를 작성할 때 두 가지 문제가 있다.

1. 보일러 플레이트(boilerplate) 코드가 많다.<br/>
데이터 클래스 생성 시 equals(), hashCode(), toString() 메서드를 만드는 등 지루한 과정을 반복해야 한다. 새 필드를 추가 할 때 클래스가 자동으로 업데이트하지 못하므로 equals() 메서드를 다시 업데이트해야 한다.

    > boilerplate code?<br/>
    > 프로그래밍에서 최소한의 변경으로 여러 곳에서 재사용되며, 반복적으로 비슷한 형태를 띄는 코드를 의미한다. 쉽게 말해 프로그래밍에서 반복되는 작업이나 패턴의 일종의 표준화된 코드를 뜻한다.
    > 
    {: .prompt-tip }

2. Person을 표현하는 클래스의 목적을 모호하게 한다.<br/>
추가되는 부가 코드로 인해 name, address라는 두 개의 String 필드를 가진 데이터 클래스라는 사실이 모호해진다.
객체 간 불변 데이터를 넘기는 것은 가장 일반적이지만 지루한 작업 중 하나다. 이러한 문제들로 인해 사소한 실수와 혼란스러운 의도가 발생할 가능성이 있었다. Java 14에 도입된 Record를 사용하여 이러한 문제를 해결할 수 있다.


### Basics
Java 14부터 이러한 반복적인 코드들을 records로 대체할 수 있다. Record는 타입과 필드의 이름만으로 작성가능한 불변의 데이터 클래스를 말한다.

`equals, hashCode, toString` 메서드와 `private, final` 필드, `public` 생성자는 Java 컴파일러에 의해 생성된다.

`Person` 클래스 선언 시 `record` 키워드를 사용하며 이름(e.g. `Person`)과 각 컴포넌트 리스트(e.g. `String name, String name`)로 구성된다.

```java
public record Person (String name, String address) { }
```

Record는 다음을 자동으로 포함한다.

- 각 컴포넌트에 대한 `private final` 필드
- 각 컴포넌트에 대한 `public read method(getter)`. 해당 메소드는 컴포넌트와 동일한 이름, 타입을 가진다.
- 레코드 컴포넌트 목록에서 시그니처를 파생하는 `public constructor`. 생성자는 해당 파라미터를 이용하여 각 private 필드를 초기화한다.
- `equals(), hashCode(), toString()` 메소드의 구현

### Constraints
- 다른 어떤 클래스도 상속받을 수 없다.
- `private final` 이외 다른 인스턴스 필드를 선언할 수 없다. 다른 모든 필드들은 `static`이어야 한다.
- Record는 암시적으로 `final`이며 `abstract`일 수 없다.

이러한 제약을 제외하면, 레코드는 일반 클래스처럼 동작한다.

- 클래스 안에 정의할 수 있고, 중첩된 Record는 암시적으로 `static`이다.
- Generic을 만들 수 있다.
- 인터페이스를 구현할 수 있다.
- new 키워드로 Record를 인스턴스화할 수 있다.
- Record Body는 static methods, static fields, static initializers, constructors, instance methods, and nested types을 선언할 수 있다.

### 관련 API
`java.lang.Class` 클래스는 Record와 관련된 새로운 메소드를 제공한다.

- `RecordComponent[] getRecordComponents()` : 레코드의 컴포넌트를 배열 형태로 반환한다.
- `boolean isRecord()` : Record로 선언 클래스는 `true`로 반환한다. `isEnum()`과 유사.

### Compact Constructor
`public constructor`가 자동 생성되지만 생성자 구현을 커스텀할 수 있다. 클래스 생성자와 달리 레코드 생성자에는 공식적인 매개 변수를 받지 않으며, 이를 `Compact Constructor`라 한다.

보통 validation을 위해 사용되며, 가능한 간단하게 구현해야 한다.

예를 들어 Person 레코드에 제공된 name, address가 `null`이 아님을 보장하도록 구현할 수 있다.
```java
public record Person(String name, String address) {
    public Person {
        Objects.requireNonNull(name);
        Objects.requireNonNull(address);
    }
}
```

또한 다른 파라미터로 새로운 생성자를 만들 수 있다.
```java
public record Person(String name, String address) {
    public Person(String name) {
        this(name, "Unknown");
    }
}
```

public constructor와 동일한 파라미터로 생성자를 만들 수도 있다. 단, 각 필드는 수동으로 초기화해야 한다.
```java
public record Person(String name, String address) {
    public Person(String name, String address) {
        this.name = name;
        this.address = address;
    }
}
```

추가적으로 compact constructor를 선언하고 해당 생성자와 동일한 파라미터로 구성된 생성자는 컴파일 에러를 발생시킨다.

따라서, 다음은 컴파일되지 않는다.

```java
public record Person(String name, String address) {
    public Person {
        Objects.requireNonNull(name);
        Objects.requireNonNull(address);
    }
    
    public Person(String name, String address) {
        this.name = name;
        this.address = address;
    }
}
```
### static 변수 및 메서드
일반적인 Java 클래스와 마찬가지로 Record에 static 변수, 메서드를 포함할 수 있다.
```java
public record Person(String name, String address) {
    public static String UNKNOWN_ADDRESS = "Unknown";
    
    public static Person unnamed(String address) {
        return new Person("Unnamed", address);
    }
}
```

### Getters
필드명과 일치하는 이름의 `getter` 메소드도 사용할 수 있다.

```java
@Test
public void givenValidNameAndAddress_whenGetNameAndAddress_thenExpectedValuesReturned() {
    String name = "John Doe";
    String address = "100 Linda Ln.";

    Person person = new Person(name, address);

    assertEquals(name, person.name());
    assertEquals(address, person.address());
}
```

다음 코드에서는 `name(), address()`가 `getter`를 의미한다.

### equals
`equals()` 메서드도 자동 생성된다. 두 객체가 같은 타입이고 모든 필드 값이 동일한 경우 `true`를 반환한다.

```java
@Test
public void givenSameNameAndAddress_whenEquals_thenPersonsEqual() {
    String name = "John Doe";
    String address = "100 Linda Ln.";

    Person person1 = new Person(name, address);
    Person person2 = new Person(name, address);

    assertTrue(person1.equals(person2));
}
```

### hashCode
`hashCode()` 메서드도 자동 생성된다. 두 객체의 모든 필드 값이 일치하는 경우 동일한 해시 코드를 반환한다.

```java
@Test
public void givenSameNameAndAddress_whenHashCode_thenPersonsEqual() {
    String name = "John Doe";
    String address = "100 Linda Ln.";

    Person person1 = new Person(name, address);
    Person person2 = new Person(name, address);

    assertEquals(person1.hashCode(), person2.hashCode());
}
```
필드 중 하나라도 다르면 리턴되는 해시 코드는 달라진다.

### toString
마지막으로 레코드의 이름과 대괄호 안에 각 필드의 이름, 값이 포함된 문자열을 반환하는 `toString` 메서드도 자동 생성한다.

```console
Person[name=John Doe, address=100 Linda Ln.]
```

## References
- https://www.baeldung.com/java-record-keyword
- https://docs.oracle.com/en/java/javase/14/language/records.html