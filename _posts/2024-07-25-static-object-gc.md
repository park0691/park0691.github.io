---
title: static object는 GC의 대상인가?
date: 2024-07-26 18:00 +0900
categories: Java
tags: java static garbage collection gc metaspace permanent generation heap
---
static 키워드는 조심해서 사용해야 한다. static 멤버는 프로그램 종료시까지 메모리에 남아있기 때문에 메모리 누수 발생으로 애플리케이션 성능 저하의 원인이 될 수 있기 때문이다.<br/>
JVM Method Area(Permanent Generation, 이하 PermGen), Metaspace를 공부하면서 의문이 생겼다. Metaspace를 알게 되기 전까지 static은 런타임 시 클래스 로더에 의해 메서드 영역에 적재되어 GC 대상이 아니라고 알고 있었다.<br/>
그런데 Java 8부터 PermGen이 사라지고 Metaspace로 대체되면서 static Object는 Heap에 저장된다고 한다. 그런데 Heap은 GC의 대상이다. 그럼 static Object는 GC의 대상이 되는 것이 맞나? GC 됨에도 static 사용에 유의해야 하는 이유는 무엇일까? 좀 혼란스러웠다.

## Java 8, Static object의 Heap 이동
openjdk JEP 122 문서에 다음과 같이 기술되어 있다.
> [Success Metrics] *Class metadata, interned Strings and class static variables will be moved from the permanent generation to either the Java heap or native memory.*<br/>
> [Description] *The proposed implementation will allocate class meta-data in native memory and move interned Strings and class statics to the Java heap.*

PermGen에 저장되던 static class는 Heap으로 옮겨갔음을 확인할 수 있다.

### PermGen은 왜 사라졌나?
PermGen은 고정된 메모리 공간으로 많은 데이터가 쌓이면 다음 메모리 부족 에러가 발생했다.
```
java.lang.OutOfMemory
```
특히 static object 중 collection에 많은 데이터가 추가되어 메모리 부족 문제가 자주 발생했다고 한다. 이를 해결하기 위해 PermGen이 제거되며, static object는 Heap으로 옮겨가게 되었다.

Java 8의 중요 변경 사항인 PermGen에서 Metaspace로 자세한 변경 사항은 정리하여 추가 포스트에 기술하겠다.

## Static Object는 GC 대상이 되는가?
결론부터 말하자면 static Object는 Heap에 저장되기 때문에 GC 대상이 된다. 이에 대해 ChatGPT에게 질문하였고 다음과 같은 답변을 얻을 수 있었다.

static 변수는 클래스 로더의 라이프 사이클에 묶여있기 때문에, 클래스가 메모리에 유지되는 한 static Object도 계속 참조된다. 다만, 경우에 따라 GC 될 수 있다.

### static 변수와 GC의 관계
#### 클래스 로더와 static 변수
- static 변수는 클래스 로더가 클래스를 메모리에 로드할 때 생성되고 클래스 로더가 언로드될 때까지 유지된다. 일반적으로 애플리케이션이 종료될 때까지 클래스가 언로드되지 않는다.
- JVM은 각 클래스 로더에 의해 로드된 클래스를 트래킹하며 클래스들이 참조하는 static 변수도 함께 트래킹한다. 따라서 클래스 로더가 언로드되지 않는 한 static 변수는 계속 존재한다.

#### null 할당 후 GC
- static 변수에 null을 할당하면 그 변수가 참조하던 객체의 참조가 제거된다. 따라서 해당 객체는 더 이상 사용되지 않으므로 GC의 대상이 된다.
- 다만 static 변수가 아닌, 다른 변수 또한 해당 객체를 참조하지 않을 때 비로소 GC의 대상이 된다. 그렇지 않으면 여전히 사용 중인 객체로 간주된다.

<br/>
공식 문서에서도 위 내용과 유사한 내용을 발견할 수 있었다.
> *Class metadata and statics are allocated in the permanent generation when a class is loaded and are garbage collected from the permanent generation when the class is unloaded.*

클래스 로더가 언로드될 때 PermGen에 대한 GC가 수행된다고 기술되어 있다. 그러나 사실 개발자가 직접 클래스 언로드를 신경 쓰는 경우는 드물다. 내장 클래스 로더는 일반적으로 클래스 JVM이 종료될 때까지 살아있으며, 사용자 정의 클래스 로더는 개발자 구현에 의존하지만 일반적인 경우 직접 하지 않는다. 따라서 PermGen에 대한 GC는 자주 발생하지 않는다고 생각하는 것이 맞다.

### static "object"
하나 놓친 것이 있었다. Heap으로 간 것은 `class static variables`다. 즉, `static object`다. static 변수 자체는 Heap이 아닌 metaspace에서 관리된다. 따라서, 클래스 로더와 생명주기를 같이하는 static 변수 자체는 GC의 영향 밖에 있는 것이다.
```java
static int a = 100;
static void test() {}
static List<Object> objList = new ArrayList<>();
```
primitive 타입 값, static 메소드, ArrayList 인스턴스의 레퍼런스는 GC와 상관 없이 메모리에 남아 있다.

## 결론
### static object는 GC의 대상이 되는가?
- Java 8부터 Heap에 저장되므로 GC의 대상이다.

### static 사용에 유의해야 하는 이유는? (메모리 관점)
- static object는 static 변수에서 레퍼런스하므로 클래스 로더와 라이프 사이클을 같이 한다. 프로그램 종료 시까지 유지되기 때문에 메모리 누수 원인이 될 수 있기 때문이다.
    - 다만 클래스가 따로 언로드되거나 `null` 할당으로 참조가 제거된 경우에는 GC 대상이다.
    - 즉, 사용하지 않게 되었을 때 static variables를 개발자가 직접 `null` 할당하지 않는 한 여전히 GC 대상이 아니기 때문에 메모리 낭비가 발생할 수 있다.
- 객체 자체는 GC 대상이 될 수 있지만 primitive 타입 또는 레퍼런스를 담는 static 변수 자체는 metaspace에 존재하며 메모리에 남아 있다. GC와 상관없이 static 변수 자체는 프로그램이 종료될 때까지 유지되므로 메모리 누수 원인이 될 수 있다.

###  과제
- PermGen, Metaspace 변경 사항 정리
- static object의 GC 발생 코드 작성해보기

## References
- https://openjdk.org/jeps/122
- https://johngrib.github.io/wiki/java8-why-permgen-removed/
- https://jonghoonpark.com/2024/04/29/java-static-object-and-gc
- ChatGPT