---
title: Class Loader API
date: 2024-08-30 18:00 +0900
categories: [Java]
tags: class loader api ClassLoader loadClass
---
자바 클래스 로더를 학습하며 실제 자바 API 코드를 확인해보고 싶었고 추가 정보들을 정리하며 분석해보고자 한다. 공부할 내용이 너무 많아 주요 메서드인 ClassLoader 클래스의 loadClass 메서드만 한 번 살펴보고 가려고 한다.

자세한 내용은 [링크](https://park0691.github.io/TIL/java/class-loader.html)에 정리하였다.

## Class Loader
### 공식  문서
다음은 공식 문서의 일부를 번역한 내용이다.

클래스를 로딩하는 책임이 있는 추상 클래스. 클래스의 `binary name`으로 파일 이름으로 바꾸고 파일 시스템으로부터 해당하는 클래스 파일을 읽어들인다.

모든 클래스 객체는 그 객체가 정의된 클래스 로더의 레퍼런스를 가진다.

배열 클래스의 객체는 클래스 로더에 의해 생성되지 않으며, 자바 런타임에 필요에 따라 자동으로 생성된다. 배열 클래스의 클래스 로더는 `Class.getClassLoader()`에 의해 리턴되며, 그 클래스 로더는 배열 요소의 타입에 대한 클래스 로더다. 배열 요소의 타입이 기본 타입이면 그 배열 클래스는 클래스 로더가 없다.

JVM이 동적으로 클래스를 로드하는 방식을 확장하기 위해 애플리케이션은 `ClassLoader`클래스의 서브 클래스를 구현한다.

클래스 로더는 클래스를 로딩하는 것 이외에도 리소스를 찾는 역할을 수행한다.

위임 모델을 사용하여 클래스와 리소스를 검색한다. 각 클래스 로더 인스턴스는 부모 클래스 로더가 있다. 클래스 또는 리소스를 찾도록 요청되면 자신이 검색하기 전에 부모 클래스 로더에게 그 요청을 위임하여 부모 클래스가 먼저 검색한다.

동시성을 지원하는 클래스 로더는 `parallel capable` 클래스 로더로 알려져 있다. 그 클래스 로더는 클래스 초기화 시점에 [`ClassLoader.registerAsParallelCapable()`](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/ClassLoader.html#registerAsParallelCapable()) 메소드를 호출하여 자기 자신을 등록해야 한다. `ClassLoader` 클래스 로더는 디폴트로 병렬 가능한 `parallel capable` 클래스 로더로 등록된다. 그러나 서브 클래스는 병렬 가능인 경우 스스로 등록해야 한다. 위임 모델이 엄격하게 계층적이지 않은 환경에서 클래스 로더는 병렬 가능 `parallel capable` 해야 한다. 그렇지 않으면 교착 상태로 이어질 수 있다. : Loader Lock이 클래스 로딩 과정동안 유지되기 때문이다. ([`loadClass()` 메소드](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/ClassLoader.html#loadClass(java.lang.String)) 참조)

### Binary names
> **Binary names**
> Any class name provided as a String parameter to methods in ClassLoader must be a binary name as defined by The Java Language Specification.
> 
> Examples of valid class names include:
>   *"java.lang.String"*
>   *"javax.swing.JSpinner$DefaultEditor"*
>   *"java.security.KeyStore$Builder$FileBuilder$1"*
>   *"java.net.URLClassLoader$3$1"*
>   
> Any package name provided as a String parameter to methods in ClassLoader must be either the empty string (denoting an unnamed package) or a fully qualified name as defined by The Java Language Specification.

클래스 로더에 전달되는 문자열로 로드할 클래스의 이름을 나타낸다. 클래스의 이름은 자바 언어 표준을 따르며, 다음 예시와 같은 형태로 표시된다. 아래 메서드에 전달되는 name 파라미터는 Binary names로 전달된다.

### 주요 메서드
- `Class<?> loadClass(String name, boolean resolve)`
	- 전달되는 name에 해당하는 클래스를 로드한다.

- `Class<?> defineClass(String name, byte[] b, int off, int len)`
	- 바이트 배열을 클래스의 인스턴스로 변환한다.

- `Class<?> findClass(String name)`
	- 전달되는 name에 해당하는 클래스를 찾는다. `loadClass()` 메서드는 부모 클래스 로더가 요청한 클래스를 찾지 못하는 경우 이 메서드를 호출한다. 클래스 로더의 부모가 클래스를 찾지 못하면 `ClassNotFoundException`을 발생시킨다.
	- ```java
	  protected Class<?> findClass(String name) throws ClassNotFoundException {
	      throw new ClassNotFoundException(name);
	  }
	  ```
	- 기본 구현체는 무조건 `ClassNotFoundException`을 발생시키도록 구현되어 있으므로 클래스 로더를 확장하는 클래스 로더는 이 메소드를 오버라이딩해야 한다.

- `ClassLoader getParent()`
  - 위임을 위해 부모 클래스 로더를 반환한다. 부트스트랩 클래스 로더는 `null`을 반환한다.

- `URL getResource(String name)`
  - 주어진 name에 해당하는 자원을 찾는다. 자원은 클래스 코드에 의해 접근되는 image, audio, text, ... 등을 말한다.

- `void resolveClass(Class<?> c)`
  - 해당 클래스를 링크한다. 만약 클래스 c가 이미 링크되어 있다면 이 메서드는 바로 리턴된다.

## loadClass() Method
### 소스 코드 분석
```java
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
                PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

> Loads the class with the specified binary name. The default implementation of this method searches for classes in the following order:<br/>
> 1. *Invoke findLoadedClass(String) to check if the class has already been loaded.*
> 2. *Invoke the loadClass method on the parent class loader. If the parent is null the class loader built into the virtual machine is used, instead.*
> 3. *Invoke the findClass(String) method to find the class.*
> 
> If the class was found using the above steps, and the resolve flag is true, this method will then invoke the resolveClass(Class) method on the resulting Class object.<br/>
> Subclasses of ClassLoader are encouraged to override findClass(String), rather than this method.<br/>
> Unless overridden, this method synchronizes on the result of getClassLoadingLock method during the entire class loading process.

- 4L : `synchronized (getClassLoadingLock(name))` 부분이 흥미롭다. 동기화 키워드를 왜 작성했을까?
	- `synchronized` 키워드는 공유 자원에 스레드간 동시 접근을 막는다.<br/>동일한 클래스가 동시에 로드될 수 있는 문제 상황을 대비하여 동기화를 걸어놓은 것이 아닐까? 클래스 로더의 원칙 중 클래스의 중복 로드가 되어서는 안 된다는 `Uniqueness Principle` 지키기 위해 작성되었다고 생각했다.
	- 동시성을 지원하는 클래스 로더의 경우 `parallel capable` 로 등록하지 않으면 데드락이 걸릴 수 있음을 알 수 있는 부분이기도 하다.

- 7L : `if (c == null) {`
	- 로드하려는 클래스가 이미 로드되지 않은 경우 클래스를 로드하는 로직을 수행한다.
	- `findLoadedClass(String)` 메소드를 이용해서 클래스가 이미 로드되었는지 체크한다.

- 10L : `if (parent != null) {`
- 11L : `c = parent.loadClass(name, false);`
	- 부모 클래스 로더가 존재하면 로딩을 위임한다. 클래스 로더의 `Delegation Principle`을 확인할 수 있는 부분이다.

- 12L ~ 13L : `} else {`
	- `parent == null`이면 부모 클래스 로더가 Bootstrap Class Loader다. 부트 스트랩 클래스 로더를  반환하는 메소드를 호출한다.

- 20L : `if (c == null) {`
- 24L : `c = findClass(name);`
	- 부모 클래스 로더에 의해 로드되지 않았다면 구현 클래스 로더가 직접 클래스를 찾는다.

- 32L : `if (resolve) {`
- 33L : `resolveClass(c);`
	- resolve 플레그가 true면 `resolveClass()` 메소드를 호출하여 링크 과정을 수행한다.

## References
- https://www.baeldung.com/java-classloaders
- https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/ClassLoader.html