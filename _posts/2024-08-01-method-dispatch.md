---
title: 호출할 메소드는 어떤 매커니즘으로 결정될까?
date: 2024-08-01 20:30 +0900
categories: Java Core
tags: dynamic dispatch static double polymorphism 다형성
---
OOP의 특징 중 하나인 다형성은 하나의 메서드나 클래스가 여러 가지 형태로 동작하는 성질을 의미합니다.

자바에서 다형성은 하나는 상속, 인터페이스를 통해 다양한 객체를 하나의 타입으로 다룰 때 나타납니다. 추상 클래스나 인터페이스를 구현한 여러 클래스에서 오버라이딩한 메소드에 따라 다양한 형태로 동작할 수 있습니다.

다형성을 통해 코드의 유지보수성을 높이고 기존 코드에 영향도를 줄여 코드의 유연성, 확장성이 좋아집니다.

그렇다면 오버라이딩된 메소드에 대한 호출은 실제로 어떤 매커니즘에 의해 처리될까요?

## Dynamic Dispatch
런타임에 실제로 호출할 메서드를 결정하는 매커니즘입니다.

이 매커니즘을 통해 런타임에 오버라이딩된 메소드에 대한 호출이 일어납니다. 컴파일 시점에 컴파일러는 인터페이스의 메소드를 호출하는 것만 알지, 실제로 어떤 메소드를 호출할지 알 수 없기 때문입니다.

Dynamic Dispatch는 참조 변수의 타입이 아닌, 실제로 참조되는 객체 타입에 따라 적절한 메서드를 호출합니다.

코드를 통해 확인해 보겠습니다.

```java
class A {
    void m1() {
        System.out.println("Inside A's m1 method"); 
    }
}

class B extends A { 
    // overriding m1() 
    void m1() { 
        System.out.println("Inside B's m1 method"); 
    }
}

class C extends A {
    // overriding m1() 
    void m1() {
        System.out.println("Inside C's m1 method"); 
    }
}

// Driver class 
class Dispatch { 
    public static void main(String args[]) { 
        // object of type A 
        A a = new A(); 
        // object of type B 
        B b = new B(); 
        // object of type C 
        C c = new C(); 
        // obtain a reference of type A 
        A ref; 
        // ref refers to an A object 
        ref = a;
        // calling A's version of m1() 
        ref.m1();
        // now ref refers to a B object 
        ref = b;
        // calling B's version of m1() 
        ref.m1();
        // now ref refers to a C object 
        ref = c;
        // calling C's version of m1() 
        ref.m1();
    }
}
```

다음 코드는 A, B, C 타입의 객체를 생성합니다. B, C 클래스의 슈퍼 클래스는 A 클래스이며<br/>B, C 클래스는 A 클래스 `m1()` 메소드를 오버라이딩합니다.

**[동작 방식]**

- 컴파일 타임
	- 컴파일러는 ref 변수의 타입을 A로 알고, A 클래스에 선언된 메서드만 알고 있습니다.
- 런타임
    - 첫 번째 `m1()` 호출 시점에 ref 변수는 A 객체를 참조하므로, A 클래스의 `m1()` 메서드를 실행합니다.
    - 두 번째 `m1()` 호출 시점에 ref 변수는 B 객체를 참조하므로, B 클래스의 `m1()` 메서드를 실행합니다.
    - 세 번째 `m1()` 호출 시점에 ref 변수는 C 객체를 참조하므로, C 클래스의 `m1()` 메서드를 실행합니다.

**[실행 결과]**

```
Inside A's m1 method
Inside B's m1 method
Inside C's m1 method
```

호출 시 실제로 참조하는 객체의 메소드가 호출되는 것을 확인할 수 있습니다.

### 장점

1. 유연성, 유지보수성
    - 객체의 실제 타입에 따라 올바른 메서드가 호출되게 하므로 코드를 더 유연하고 확장 가능하게 만듭니다.

2. 다형성 구현
    - 다형성을 통해 동일한 인터페이스 또는 부모 클래스를 공유하는 객체들이 다양한 동작을 수행할 수 있도록 구현할 수 있습니다.

### 데이터 멤버의 런타임 다형성

자바에서는 메서드만 재정의할 수 있으므로 데이터 멤버 변수는 런타임 다형성을 보장하지 않습니다.

코드를 통해 확인해 보겠습니다.

```java
class A { 
    int x = 10; 
} 

class B extends A { 
    int x = 20; 
} 
  
// Driver class 
public class Test { 
    public static void main(String args[]) { 
        A a = new B();
  
        // Data member of class A will be accessed 
        System.out.println(a.x); 
    } 
} 
```

**[실행 결과]**

```java
10
```

부모 클래스인 A와 B는 모두 멤버 변수 `x`를 가지고 있습니다. A 타입의 a 변수에 B 타입의 객체를 대입했으나, 다형성이 보장되지 않으므로 항상 부모 클래스 A의 데이터를 참조합니다.

이와 대비되는 다른 매커니즘도 존재합니다.

## Static Dispatch

컴파일 타임에 메소드 호출이 결정되는 매커니즘입니다.

컴파일러가 소스 코드를 분석하여 호출할 메서드를 결정하므로 런타임에 추가적으로 호출할 메서드를 결정하지 않습니다. 이 매커니즘은 메서드 오버로딩과 관련됩니다.

코드를 통해 확인해 보겠습니다.

```java
class Printer {
    void print(String message) {
        System.out.println("String message: " + message);
    }

    void print(int number) {
        System.out.println("Integer number: " + number);
    }
}

public class Main {
    public static void main(String[] args) {
        Printer printer = new Printer();
        
        printer.print("Hello");   // "String message: Hello" 출력
        printer.print(10);        // "Integer number: 10" 출력
    }
}
```

**[동작 방식]**

- 컴파일러는 매개변수의 타입을 기준으로 적절한 메서드 결정합니다.
- 첫 번째 print() 메소드 호출 시 컴파일러는 매개변수의 타입이 String으로 정의된 print(String message) 메서드를 선택합니다.
- 두 번째 print() 메소드 호출 시 컴파일러는 매개변수의 타입이 int로 정의된 print(int number) 메서드를 선택합니다.

### 장점

- 런타임 오버헤드 감소
    - 컴파일 타임에 메서드 호출이 결정되므로 런타임에 추가적인 메서드 결정 과정이 없어 다이나믹 디스패치에 비해 런타임 메서드 호출의 오버헤드를 줄일 수 있습니다.
- 타입 안전성
    - 컴파일러가 호출할 메서드를 결정할 때 매개변수의 타입을 검증하므로 잘못된 타입의 매개변수를 전달하는 실수를 줄일 수 있습니다.
- 오버로딩을 통한 코드 가독성, 유지보수성
    - 같은 이름의 메서드를 여러 가지 매개변수 시그니처로 정의하여 메서드 이름의 일관성을 지키면서 다양한 입력을 처리할 수 있습니다.

## Double Dispatch

Dynamic Dispatch를 두 번 하는 것을 의미합니다.

코드를 통해 확인해 보겠습니다.

```java
public class DynamicDispatch {
    interface Post {
        void postOn(SNS sns);
    }

    static class Text implements Post {
        @Override
        public void postOn(SNS sns) {
            sns.post(this);
        }
    }

    static class Picture implements Post {
        @Override
        public void postOn(SNS sns) {
            sns.post(this);
        }
    }

    interface SNS {
        void post(Text post);
        void post(Picture post);
    }

    static class Facebook implements SNS{
        @Override
        public void post(Text post) {
            System.out.println("text - facebook");
        }

        @Override
        public void post(Picture post) {
            System.out.println("picture - facebook");
        }
    };

    static class Twitter implements SNS{
        @Override
        public void post(Text post) {
            System.out.println("text - twitter");
        }

        @Override
        public void post(Picture post) {
            System.out.println("picture - twitter");
        }
    };

    static class GooglePlus implements SNS {
        @Override
        public void post(Text post) {
            System.out.println("text - googleplus");
        }

        @Override
        public void post(Picture post) {
            System.out.println("picture - googleplus");
        }
    }

    public static void main(String[] args) {
        List<Post> posts = Arrays.asList(new Text(), new Picture());
        List<SNS> sns = Arrays.asList(new Facebook(), new Twitter());

        posts.forEach(p -> sns.forEach(p::postOn));
    }
}

```

**[수행 결과]**

```java
text - facebook
text - twitter
picture - facebook
picture - twitter
```

p.postOn() 메서드를 호출할 때 Dynamic Dispatch가 한 번 일어나고<br/>(POST 구현 클래스의 메소드가 런타임에 결정)

호출된 postOn() 메서드 내부에서 sns.post() 메서드를 호출할 때 Dynamic Dispatch가 한 번 더 일어납니다.<br/>(SNS 구현 클래스의 메소드가 런타임에 결정)

## References
- https://www.geeksforgeeks.org/dynamic-method-dispatch-runtime-polymorphism-java/
- https://kdg-is.tistory.com/entry/JAVA-%EB%8B%A4%EC%9D%B4%EB%82%98%EB%AF%B9-%EB%A9%94%EC%86%8C%EB%93%9C-%EB%94%94%EC%8A%A4%ED%8C%A8%EC%B9%98%EB%9E%80
- https://www.youtube.com/live/s-tXAHub6vg?si=kHHyZiOjgfcDDStw