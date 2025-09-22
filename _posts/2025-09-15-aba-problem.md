---
title: CAS 연산의 숨겨진 함정 - ABA 문제
date: 2025-09-15 12:00 +0900
categories: Concurrency
tags: aba, problem
---

스레드를 멈추지 않고 하드웨어 수준의 원자적 연산을 통해 동시성을 제어하는 Lock-Free 알고리즘은 병렬 프로그래밍의 핵심 기술이다.

Lock-Free 알고리즘은 다음 포스트에서 다루었다.

[https://park0691.github.io/posts/java-cas-atomic/](https://park0691.github.io/posts/java-cas-atomic/)

하지만 이 기법의 이면에는 **ABA 문제**라는 치명적인 함정이 숨어있다.

변수의 값이 `A`에서 `B`로, 다시 `A`로 돌아왔을 때 CAS(Compare-And-Swap) 연산이 '아무 것도 변하지 않았다' 고 착각하여 데이터의 일관성을 깨뜨리는 문제가 발생한다. 값은 같지만 그 사이에 다른 변경이 있었던 상황은 찾기 힘든 버그를 유발하여 Lock-Free 자료구조를 망가뜨릴 수 있다.

이 포스트에서는 C++로 구현된 Lock-Free 스택 예제를 통해 ABA 문제가 실제로 어떻게 데이터 구조를 손상시키는지 살펴보고, 이를 해결하기 위한 아이디어들을 함께 알아본다.

## ABA Problem
Lock-Free 알고리즘에서 ABA 문제란, 특정 값이 `A`에서 `B`로 바뀌었다가 다시 `A`로 돌아왔을 때, CAS(Compare-And-Swap) 연산이 '값이 변경되지 않았다'고 착각하는 현상을 말한다.

값 자체는 같지만 그 중간에 다른 연산이 개입했음에도 CAS는 이를 감지하지 못하고 업데이트를 성공시켜 데이터의 일관성을 깨뜨릴 수 있다.

예를 들어, 한 스레드가 메모리 주소 `M`의 값이 포인터 `A`임을 확인했다고 가정하자. 이 스레드는 `M`의 값을 다른 것으로 바꾸기 위해 `CAS(M, A, newValue)`를 시도할 것이다. 하지만 그 사이, 다른 스레드가 끼어들어 다음과 같이 값을 변경한다.

1. `M`의 값 `A`를 `B`로 변경한다.

2. 이후, 메모리 재사용 등의 이유로 `M`의 값을 다시 `A`로 되돌린다.

```
M = A -> C
M = B -> D
M = A -> E
```

첫 번째 스레드가 다시 실행되어 CAS를 시도하면, `M`의 값은 여전히 `A`이므로 연산은 성공한다. 하지만 `A`가 가리키는 실제 내용은 이미 변경되었을 수 있어, 데이터 구조의 일관성이 깨질 위험이 있다.

이 문제는 **<u>메모리 재사용(Memory Reuse)</u>**과 깊은 관련이 있다. 현대 메모리 관리자는 성능 최적화를 위해 해제된 메모리를 적극적으로 재사용하는데, 이 과정에서 ABA 문제가 발생할 수 있다. CAS는 메모리 주소의 '값'만 비교할 뿐, 그 값이 그 사이에 거친 다른 맥락은 모르기 때문이다.

특히 메모리 재사용은 현대 대부분의 메모리 관리자가 사용하는 최적화 방법이기 때문에 동시/병렬 프로그래밍과 Lock-Free 알고리즘을 구현할 때 꼭 고려해야 하는 사항이다.

예를 들어, C++로 구현된 Lock-Free Stack 구현을 보자

```c
class Stack {
    volatile Obj* top_ptr;

    Obj* Pop() {
        while(1) {
            Obj* ret_ptr = top_ptr;          //(1)
            if (!ret_ptr) return NULL;       //(2)
            Obj* next_ptr = ret_ptr->next;   //(3)
            if (CompareAndSwap(top_ptr, ret_ptr, next_ptr)) //(4)
                return ret_ptr;
        }
    }

    void Push(Obj* obj_ptr) {
    ... 
}
```

스택에 `C` ← `B` ← `A` 순서로 3개의 데이터가 있다고 하자. 스택의 `top`은 A이고, `next`는 B다.

스레드가 `(3)`까지 수행했는데 다른 스레드가 `(3)`과 `(4)` 사이에 끼어들어 `pop`을 2번 수행하고, 다시 `push`를 1번 수행했다고 가정하자.

![images](https://1drv.ms/i/c/9251ef56e0951664/IQR-IjPTuZ2jTKeruGDnIhpGAQjojslSEBFDlvRzWixZfck)
_출처 : https://shchukin-alex.medium.com/lock-free-in-swift-aba-problem-4d68622164da_

1. 스레드 0: `A`를 `pop` 시도
    - 스택의 `top`은 `A`임을 기억한다.
2. 스레드 0: 잠시 중단 (Preemption)
    - `CAS` 연산인 `(4)`를 수행하기 직전, OS 스케줄러에 의해 잠시 멈춘다.
    - 이 때, 스레드 0은 `(3)`까지 수행한 상태 (`ret_ptr`, `next_ptr`에는 이미 값이 들어 있음) 이다. (`ret_ptr : A, next_ptr : B`)
    - `next_ptr`은 `top_ptr`이 변경됬다면 폐기되어야 하며, CAS 연산은 그 역할을 한다.
3. 스레드 1: 개입하여 스택 조작
    - 그 사이에 스레드 1이 실행된다.
    - 스레드 1은 `A`를 `pop` 한다. (스택 : `C` ←  `B(top)`)
    - 스레드 1은 `B`도 `pop` 한다. (스택 : `C(top)`)
    - 스레드 1은 다시 `A`를 `push` 한다. (스택 : `C` ← `A(top)`)
4. 스레드 0: 다시 실행
    - 스레드 0이 다시 실행된다.
    - `CAS` 연산인 `(4)`를 수행한다.
    - 현재 스택의 `top`은 `A`이므로 예상한 값과 일치하여 `CAS` 연산은 성공한다.

**[데이터 구조 손상]**<br/>
2번 `pop` 했기 때문에 위쪽 2개(`A`,`B`)는 메모리 할당이 해제되므로, 메모리 관리자는 최적화를 위해 그 공간을 재사용할 수 있다.

스레드 1이 `push`할 때 스레드 0의 `ret_ptr`에 저장된 공간인 `A`를 재사용하면 스레드 0이 실행 재개했을 때 CAS는 성공한다.

이 때, 스레드 0은 스택의 `top`을 자신이 예전에 기억했던 다음 노드(`next_ptr`)인 `B`로 설정한다.

그러나 `B`는 이미 스레드 1에 의해 스택에서 제거된, 유효하지 않은 노드다.

결과적으로 스택의 포인터는 유효하지 않은 메모리를 가리키게 되어 데이터 구조가 손상되고, 프로그램은 심각한 오류에 빠질 수 있다.

### Solution : DCAS, Double Compare-And-Swap
말 그대로 두 번 CAS를 하는 것이다. 위의 예시에서 스택의 `top`과 `next` 포인터 둘을 동시에 체크하여 ABA 문제를 해결할 수 있다.

즉, 스택의 `top`이 여전히 내가 아는 노드 `A`를 가리키고, 동시에 그 노드 `A`의 다음 노드도 내가 알던 노드 `B`가 맞을 때 `CAS` 연산이 성공한다.

**[하드웨어 미지원]**<br/>
DCAS는 개념적으로 매우 강력하지만, 대부분의 최신 CPU(x86, ARM 등)는 DCAS를 하드웨어 명령어로 지원하지 않는다.

CAS는 하드웨어 수준에서 지원되어 매우 빠르고 효율적이지만, DCAS는 소프트웨어적으로 복잡하게 구현해야만 한다. 이는 성능 이점을 상쇄시키므로 DCAS는 널리 사용되지 못하고 이론적인 개념으로 남는 경우가 많다.

### Solution : Versioning, Stamping
값을 비교할 때, 값 자체뿐만 아니라 버전 정보까지 함께 비교하는 방식이다.

- 값이 변경될 때마다 버전을 1씩 증가시킨다.
- CAS 연산은 `(값, 버전)` 쌍을 비교한다.

## References
- https://blog.naver.com/jjoommnn/130040068875
- https://shchukin-alex.medium.com/lock-free-in-swift-aba-problem-4d68622164da
- Gemini 2.5 Pro