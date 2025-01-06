---
title: 스프링 @Transactional 활용 시 주의점
date: 2025-01-03 12:00 +0900
categories: Spring
tags: spring, transaction, transactional
---

개발할 때 서비스 계층에 `@Transactional` 어노테이션을 많이 사용한다. 연산들을 원자 단위로 수행되게 하기 위해 묶어주고, 예외 발생시 롤백해 준다는 정도로 알고 습관적으로 사용할 때가 많았다.

트랜잭션의 중첩 이슈와 내부 옵션들에 대한 호기심이 생겨 이번 기회에 `@Transactional` 어노테이션의 동작과 옵션, 주의점 등을 알아보고자 한다.

## @Transactional
클래스나 메서드 단위로 붙이며, 해당 범위 내 메서드가 트랜잭션이 되도록 보장한다.

연산이 고립되어 다른 연산과의 혼선으로 잘못된 값을 가져오는 것 방지하며, 연산의 원자성이 보장되어, 도중에 실패할 경우 내부 변경 사항이 COMMIT 되지 않는다.

> 트랜잭션
> DBMS 또는 유사한 시스템에서 상호작용의 단위로 더 이상 쪼개어지지 않고 한 번에 수행되어야 하는 연산을 말한다.
{: .prompt-tip }

### 작동 원리
@Transactional이 메서드에 붙으면 스프링은 해당 메서드에 대한 프록시를 만든다.

트랜잭션의 시작, 연산 종료 시 커밋 로직을 프록시를 생성해 호출할 메서드의 앞, 뒤에 추가하며 예외 발생시 롤백 처리를 추가한다.

이러한 원리는 AOP에 바탕을 두고 설계되었기 때문에 트랜잭션 AOP로 부르기도 한다.

### 우선 순위
```
클래스 메소드 -> 클래스 -> 인터페이스 메소드 -> 인터페이스
```

클래스 메서드에 선언된 트랜잭션의 우선 순위가 가장 높고, 인터페이스에 선언된 트랜잭션의 우선 순위가 가장 낮다.

### 옵션
#### isolation, 격리 레벨
```java
@Transactional(isolation = Isolation.DEFAULT)
public void addBook() throws Exception {
    ...
}
```
- `DEFAULT` : DB의 격리 레벨을 따른다.
- `READ_UNCOMMITTED`
- `READ_COMMITTED`
- `REPEATABLE_READ`
- `SERIALIZABLE`

격리 레벨에 대한 내용은 다음에서 참조한다. [TIL : 격리 수준](https://park0691.github.io/TIL/database/transaction.html#%E1%84%80%E1%85%A7%E1%86%A8%E1%84%85%E1%85%B5-%E1%84%89%E1%85%AE%E1%84%8C%E1%85%AE%E1%86%AB)

#### propagation, 전파 속성
```java
@Transactional(propagation = Propagation.REQUIRED)
public void addBook() throws Exception {
    ...
}
```
기존 트랜잭션이 진행 중인데 추가적인 트랜잭션을 어떻게 진행할 지 결정한다.

##### REQUIRED
- 기존 트랜잭션이 있다면 그 트랜잭션을 따르고, 없으면 새 트랜잭션을 시작한다.
- 스프링의 default 전파 속성
  
##### REQUIRES_NEW
- 항상 새 트랜잭션을 시작한다. (외부 트랜잭션과 내부 트랜잭션을 완전히 분리)
- 기존 트랜잭션과 상관없이 새로운 트랜잭션을 만들어 독립적으로 동작한다.
  - 부모 트랜잭션이 롤백되더라도 자식 트랜잭션은 롤백되지 않고, 자식 트랜잭션이 롤백되더라도 부모 트랜잭션은 롤백되지 않는다. (롤백시 서로의 트랜잭션에 영향을 주지 않는다.)

> 내부 트랜잭션이 생성된만큼 DB 커넥션이 생성되기 때문에, 내부 트랜잭션이 생성되면 한 HTTP 요청에 대해 여러 DB 커넥션을 사용한다. 디비 커넥션을 고갈시킬 가능성이 있으므로 성능에 유의해서 사용해야 한다.


##### NESTED
- 기존 트랜잭션이 있으면 중첩 트랜잭션을 만들고, 없으면 새로운 트랜잭션을 만든다.
- <u>중첩 트랜잭션은 부모 트랜잭션 (커밋/롤백)의 영향을 받는다.</u>
  - 부모 트랜잭션에서 롤백되면 중첩 트랜잭션도 함께 롤백됨
  - 부모 트랜잭션이 완료될 때 커밋됨
- 중첩 트랜잭션의 롤백은 부모 트랜잭션에 영향을 주지 않는다.
  - 중첩 트랜잭션이 롤백되어도 부모 트랜잭션은 커밋됨

| 특징 | 중첩 트랜잭션(NESTED) | 독립 트랜잭션(REQUIRES_NEW) |
| --- | --- | --- |
| 새 트랜잭션 생성 여부  | 부모 트랜잭션의 일부로 SAVEPOINT 생성 | 완전히 새로운 트랜잭션 생성 |
| 부모 트랜잭션에 영향 | 부모 트랜잭션 커밋/롤백에 종속적 | 부모 트랜잭션과 독립적 |
| 롤백 볌위 | 중첩 트랜잭션의 SAVEPOINT 까지 롤백 | 독립 트랜잭션 전체 롤백 |
| 커넥션 | 부모 트랜잭션과 같은 트랜잭션 사용 | 부모 트랜잭션과 다른 트랜잭션  사용 |

- DBMS의 `savepoint` 기능을 사용하므로 DB 드라이버가 이를 지원하는지 확인이 필요하며, JPA에서는 사용 불가하다.

###### 어떻게 SAVEPOINT를 이용하는가?
- `savepoint`는 트랜잭션 처리 중 롤백을 위한 특정 지점을 가리키는 개념
- 트랜잭션을 처리하다가 오류가 발생했을 때 `ROLLBACK TRAN` 문을 사용하여 전체 트랜잭션이 취소되는 것을 막을 수 있는 방법으로 `ROLLBACK TRAN SAVEPOINT_NAME` 형식을 사용하여 특정 시점까지 트랜잭션을 취소할 수 있게 한다.

```sql
CREATE TABLE tran4 (id int, trancnt int)
		
BEGIN TRAN
    INSERT TRAN4 VALUES(1,@@trancount)
    SAVE TRANSACTION tran_step1 	    -- 트랜잭션 SAVEPOINT 정의
        INSERT tran4 VALUES(2,@@TRANCOUNT)
    ROLLBACK TRAN tran_step1		    -- SAVEPOINT 시점까지만 취소
    INSERT tran4 VALUES(3,@@trancount)
COMMIT TRAN

SELECT * FROM tran4
```
다음은 MS-SQL 환경에서 작성한 쿼리이다.

| id |  trancnt |
| --- | --- |
| 1  | 2  |
| 3  | 2  |

수행 결과 2번에 대한 데이터만 롤백 처리되었음을 알 수 있다.

##### MANDATORY
- 트랜잭션이 반드시 필요
- 기존 트랜잭션이 없으면 `IllegalTransactionStateException` 예외 발생
- 기존 트랜잭션이 있으면 그 트랜잭션에 참여

##### SUPPORTS
- 기존 트랜잭션이 있으면 그 트랜잭션에 참여, 없으면 트랜잭션 없이 진행한다.

##### NOT_SUPPORTED
- 트랜잭션을 사용하지 않고 처리한다.
- 기존 트랜잭션이 없어도 새로운 트랜잭션을 생성하지 않는다.
- 기존 트랜잭션이 있으면 그 트랜잭션을 보류시키고 트랜잭션 없이 메소드를 수행한 후 기존 트랜잭션을 재개한다.

##### NEVER
- 트랜잭션을 사용하지 않도록 강제한다. (기존 트랜잭션을 허용 X)
- 기존 트랜잭션이 있으면 `IllegalTransactionStateException` 예외 발생

#### readOnly
- 트랜잭션을 읽기 전용으로 설정한다.
  - INSERT, UPDATE, DELETE 연산 시 예외 발생
- JPA의 경우 해당 옵션을 `true`로 설정하면 트랜잭션이 커밋되어도 영속성 컨텍스트를 플러시하지 않는다 .(`FlushMode = NEVER`) 플러시할 때 수행되는 엔티티의 스냅샷 비교 로직이 수행되지 않으므로 성능을 향상시킬 수 있다.

#### rollbackFor
```java
@Transactional(rollbackFor = Exception.class)
public void addBook() throws Exception {
    ...
}
```
- 특정 예외 발생시 강제로 롤백한다.

#### noRollbackFor
```java
@Transactional(noRollbackFor = Exception.class)
public void addBook() throws Exception {
    ...
}
```
- 특정 예외 발생시 롤백 처리하지 않는다. (예외 무시)

#### timeout
```java
@Transactional(timeout = 10)
public void addBook() throws Exception {
    ...
}
```
- 지정한 시간 내에 해당 메소드의 수행이 완료되지 않을 경우 롤백한다.
- 디폴트 값은 -1 (no timeout)

### 트랜잭션 범위의 영속성 컨텍스트
![persistence](https://1drv.ms/i/c/9251ef56e0951664/IQSC4RrO3kp2S4Gde30O-wm1AUprIjTHpa77jvO3QRFmN6c)
_자바 ORM 표준 JPA 프로그래밍, 김영한 저_

해당 코드 내의 메서드를 호출할 때 영속성 컨텍스트가 생긴다.

영속성 컨텍스트는 트랜잭션 AOP가 트랜잭션을 시작할 때 생겨나고, 메서드가 종료되어 트랜잭션 AOP가 트랜잭션을 커밋하면 영속성 컨텍스트가 `flush` 되어 해당 내용을 반영한다. 이후 영속성 컨텍스트도 종료된다.

- 같은 트랜잭션 내에서 여러 EntityManager를 쓰더라도, 같은 영속성 컨텍스트를 사용한다.
- 같은 EntityManager를 쓰더라도, 트랜잭션이 다르면 다른 영속성 컨텍스트를 사용한다.

### 주의 사항
#### Checked Exception(IOException, SQLException, ...)은 관리하지 않는다.
Unchecked Exception(RuntimeException) 발생 시 트랜잭션을 롤백하지만 Checked Exception 발생 시 롤백하지 않는다.

> Checked Exception 발생 또는 예외가 없을 때, Commit<br/>
> Unchecked Excpeion 발생 시, Rollback
> 
{: .prompt-warning }

`rollbackFor` 옵션으로 특정 예외를 지정하면, Checked Exception 발생 시 롤백 처리할 수 있다.

#### synchronized와 같이 사용할 때 갱신 손실 문제 해결 불가
@Transactional은 AOP가 적용되기 때문에 프록시로 구동되기 때문이다.

AOP로 인해 프록시 객체에서 실제 객체 메소드의 실행이 끝나고 트랜잭션이 커밋되기 직전에 다른 스레드가 데이터를 읽기 때문에 갱신 손실 문제를 해결할 수 없다.

자세한 내용은 다음 포스트에서 다루겠다.

#### non-public (protected, private) 메소드에 적용할 수 없다.
non-public 메소드에는 프록시를 적용할 수 없기 때문이다.

스프링 AOP의 디폴트 프록시는 CGLIB 기반으로 대상 클래스를 상속 받아 프록시를 생성하기 때문에 private 메소드는 상속할 수 없어서 프록시 적용이 어렵다.

> **protected는 상속 관계에서 접근 가능하기 때문에 적용되어야 하지 않을까?**
>
> 1. 일관성 유지
>   - 스프링은 JDK 동적 프록시와 CGLIB 프록시를 모두 지원한다.
>   - JDK 동적 프록시는 인터페이스 기반으로 동작하므로, protected 메서드는 프록시로 감쌀 수 없다.
>   - 두 방식 간 동작의 일관성을 유지하기 위해 스프링은 기본적으로 public 메서드만 트랜잭션 처리 대상으로 간주한다.
> 2. 관례에 따른 설계
>   - @Transactional은 주로 서비스 계층의 공개 API에 적용된다.
>   - protected나 private 메서드에 트랜잭션을 적용하는 요구는 상대적으로 드문데, 이를 강제로 허용하면 불필요한 복잡성이 추가될 수 있기 떄문이다.
{: .prompt-tip }

**[private method에도 프록시를 적용하려면?]**
- 스프링 트랜잭션의 모드를 `AspectJ` 모드로 변경한다.

> 스프링 트랜잭션 모드
> - AdviceMode.PROXY
>   - default mode로 Spring AOP의 프록시 매커니즘 (JDK Dynamic 또는 CGLIB) 사용
>   - public method에만 적용된다.
> - AdviceMode.ASPECTJ
>   - AspectJ 프레임워크를 이용하여 트랜잭션 경계를 처리
>   - 바이트코드 조작을 통해 프록시를 사용하지 않고도 AOP를 구현한다.
>   - 직접 바이트코드를 주입하기 때문에 
>     - non-public method에도 트랜잭션을 적용할 수 있다.
>     - 자신의 클래스 내부 메소드를 호출해도 트랜잭션 코드가 포함된 메서드를 호출한다.
>   - Proxy 기반 모드에 비해 더 세밀한 트랜잭션 제어가 가능하다.
> - 이 외에도 `Reactive Transaction(비동기 리액티브 트랜잭션 처리), ChainedTransactionManager(다중 트랜잭션 매니저 처리), Global Transaction (JTA, 분산 트랜잭션 처리), Manual Mode(프로그래밍 방식 트랜잭션 처리)` 등 다양한 모드가 존재한다.
{: .prompt-tip }

#### Proxy 외부에서 접근해야만 AOP가 적용된다.
- 내부 호출 (`self-invocation`)에서는 프록시를 적용할 수 없다.

`@Transactional`이 적용되지 않은 메소드에서 `@Transactional`이 적용된 메서드를 호출하는 경우를 들 수 있다.

예시 코드를 보자

```java
@Service
public class BookImpl implements Books {
    public void addBooks(List<String> bookNames) {
        bookNames.forEach(bookName -> this.addBook(bookName));
    }
    
    @Transactional
    public void addBook(String bookName) {
        Book book = new Book(bookName);
        bookRepository.save(book);
        book.setFlag(true);
    }
}
```

위와 같은 코드에서 `@Transactional`이 적용되지 않은 `addBooks()`  메소드를 호출하는 경우 해당 메소드 내부에서 호출되는 `@Transactional`을 적용한 `addBook()` 메서드를 호출하더라도 트랜잭션 AOP는 동작하지 않는다.

따라서 `book.setFlag(true);` 코드로 엔티티의 변경을 수행했으나 엔티티의 변경이 수행되지 않는다. 트랜잭션 커밋 처리되지 않아 해당 엔티티의 변경 감지가 동작하지 않기 때문이다.

**[왜 트랜잭션 AOP가 동작하지 않을까?]**

`addBooks()` 메소드 내부에서 트랜잭션 처리된 프록시 객체의 `addBook()` 메소드를 호출하지 않고, <u>실제 객체의 메소드를 호출하기 때문</u>이다.

**[해결 방법]**
1. 가급적이면 트랜잭션 처리된 메소드는 클래스 내부에서 호출하지 않고 외부에서 호출하는 것을 권장한다.
- 클래스 내부에서 호출하는 @Transactional 메소드를 다른 클래스로 분리하는 방법을 고려한다.

2. Self Autowired 사용

	굳이 내부적으로 사용한다면, 자기 자신의 프록시 객체를 호출하도록 한다.

    ```java
    @Service
    public class BookImpl implements Books {
        // 생성자 주입 시 순환 참조 에러 발생하므로 주의. @Autowired 사용
        @Autowired
        private Books self;

        public void addBooks(List<String> bookNames) {
            bookNames.forEach(bookName -> self.addBook(bookName));
        }

        @Transactional
        public void addBook(String bookName) {
            Book book = new Book(bookName);
            bookRepository.save(book);
            book.setFlag(true);
        }
    }
    ```

3. `TransactionTemplate` 활용

## References
- gpt4o
- 자바 ORM 표준 JPA 프로그래밍, 김영한 저
- https://flambeeyoga.tistory.com/entry/Transactional-%EC%82%AC%EC%9A%A9-%EC%8B%9C-%EC%A3%BC%EC%9D%98%EC%A0%90
- https://velog.io/@ddongh1122/Spring-Transactional-%ED%81%B4%EB%9E%98%EC%8A%A4-%EB%82%B4%EB%B6%80-%ED%98%B8%EC%B6%9C-%EB%AF%B8%EC%9E%91%EB%8F%99-%EC%9D%B4%EC%8A%88
- http://ankyu.entersoft.kr/lecture/ms_sql/09_transaction03.asp
- https://mangkyu.tistory.com/269