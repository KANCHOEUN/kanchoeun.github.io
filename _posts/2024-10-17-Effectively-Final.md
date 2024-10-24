---
title: Effecitvely Final
# description: Examples of text, typography, math equations, diagrams, flowcharts, pictures, videos, and more.
author: kancho
date: 2024-10-17 11:56:00 +0500
categories: [Programming Languages, Java]
tags: [Java]
pin: true
# math: true
# mermaid: true
---

## 개요

![effectively_final](/assets/img/posts/effectively_final.png)
<br/>

Java로 [알고리즘 문제](https://github.com/KANCHOEUN/Algorithm/blob/master/CodeTree/Silver/%EB%B9%99%EC%82%B0%EC%9D%98%EC%9D%BC%EA%B0%81.md)를 푸는 도중 람다식 내부에서 람다식 외부의 지역 변수를 사용하고자 했으나, 사용할 수 없다는 에러가 발생했다. 찾아보니 Java의 Effectively Final이라는 개념에 의해 발생한 에러라서 이에 대해 정리해보았다.

<br/>

## 정의
### Effectively Final
> Java에서 final로 선언되지 않았지만, 초기화된 이후 참조가 변경되지 않아 final처럼 동작하는 것

`Effectively Final` 은 Java 8에 도입되었는데, <span style='color:#f7b731'>익명 클래스</span>(Anonymous Class) 또는 <span style='color:#f7b731'>람다 표현식</span>(Lambda Expression)이 사용된 코드에서 쉽게 찾아볼 수 있다.

익명 클래스 혹은 람다식에서는 외부 지역 변수가 `final` 로 선언되었거나, 선언된 후 참조가 변경되지 않는 `effectivley final` 인 경우에만 접근이 가능하다.

<br/>

#### effectively final이 되기 위한 3가지 조건
[Java 언어 스펙](https://docs.oracle.com/javase/specs/jls/se8/html/jls-4.html#jls-4.12.4) 기준 다음과 같은 조건을 만족한 지역 변수를 effectively final이라고 한다.

1. `final` 로 선언되지 않았다.
2. 초기화 후 다시 할당되지 않았다. (단, 초기화 없이 선언된 후, 단 한 번 할당하는 것은 무방하다.)
3. prefix 혹은 postfix에 증감 연산자가 사용되지 않았다.

<br/>

```java
int number; // 1) final로 선언되지 않았음
number = 10; // 초기 할당
// number = 20; -> 2) 재할당 불가능
// number++; -> 3) 증가 연산자 사용 불가능
Runnable r = () -> System.out.println(number); // 람다 함수
r.run();
```

<br/>

## 왜 람다에서 `(effectively) final` 을 써야하는가
이를 이해하기 위해서는 먼저 다음 두 가지에 대한 이해가 필요하다

- 지역 변수와 람다 표현식이 어느 Memory 영역에 저장되는가
- 람다 캡처링 (Lambda Capturing)

### 1) 지역 변수와 람다 표현식은 어느 영역에 저장되는가?
JVM의 Memory 영역은 크게 Stack 영역과 Heap 영역으로 나뉜다.

- Stack 영역
	- Stack은 각 스레드마다 하나씩 존재하며, 스레드가 시작될 때 할당된다.
	- 메서드 호출 시마다 각각의 스택 프레임(해당 메서드만을 위한 공간)이 생성되고, 메서드 안에서 사용되는 값들을 저장하고, 메서드 종료 시 프레임 별로 소멸된다.
- Heap 영역
	- 모든 스레드가 공유하며, 런타임 시 동적으로 할당하여 사용하는 영역이다.
	- new 연산자로 생성되는 클래스/인스턴스 변수, 배열 타입 등 Reference 타입이 저장되는데, 이러한 타입의 객체를 참조하는 변수는 Stack 영역에 저장된다. 만약 참조하는 변수나 필드가 없다면, GC에 의해 자동으로 제거된다.

<br/>

```java
int number = 10; // 지역 변수
Runnable r = () -> System.out.println(number); // 람다 함수
r.run();
```

<br/>

`number` 와 같은 지역 변수는 Stack 영역에 저장되어, 메서드 종료 시 소멸되어 접근이 불가능하다. 반면 `() -> System.out.println(number)` 와 같은 람다 표현식은 런타임에 함수형 인터페이스의 구현체로 변환되고, 해당 구현체의 인스턴스가 Heap 영역에 저장된다.

메소드 내 지역 변수를 참조하는 람다식을 리턴하는 메소드가 있다고 가정했을 때, 해당 메소드가 종료되면 메소드 내 지역 변수는 스택에서 제거된다. 따라서 이후 람다식이 실행될 때 지역 변수를 참조할 수 없는 문제가 발생한다.

이러한 문제를 해결하기 위해 람다 캡처링이 필요하다.

### 2) 람다 캡처링 (Lambda Capturing)
람다 표현식에서 <span style='color:#f7b731'>외부에 정의된 지역 변수</span>를 사용할 때, <span style='color:#f7b731'>내부에서 사용할 수 있도록 복사본을 생성하는 것</span>을 말한다.

```java
public class Test {
	public void test() {
		int count = 0;
		Runnable r = () -> System.out.println(count);
	}
}
```

외부 지역 변수를 그대로 사용하지 않고, 복사본을 사용하는 것은 이해했다. 그렇다면 왜 외부 지역 변수는 `final` 혹은 `effectively final` 이어야 할까?

### 3) `final` 혹은 `effectively final` 이어야만 하는 이유

1. 만약 람다식이 실행되는 동안 외부 변수가 변경 가능하다면, 그 변화를 추적하고 동기화하는 것이 필요한데, 우리는 람다 캡처링을 통해 복사본을 만들었기 때문에 불가능하다.
2. 멀티 스레드 환경에서 한 스레드가 외부 변수를 수정하게 된다면, 다른 스레드가 이를 반영할 수 있어야 하는데, 이 또한 복사본을 만들었기 때문에 불가능하다.

따라서 람다식 내부에서 복사해온 외부 지역 변수 값이 변경되지 않은 최신값임을 보장하기 위해 외부 지역 변수는 `final` 혹은 `effectively final` 이어야 한다.

> Java 7에서는 외부 지역 변수가 final인 경우에만 접근이 가능하도록 강제했었는데, Java 8부터 effectively final인 경우에도 접근이 가능하도록 바뀌게 되었다고 한다.

<br/>

## 정리
정리하자면 다음 3가지 이유로 인해 람다 표현식에서는 final 혹은 effectively final 이어야 한다.

1. 지역 변수는 stack 영역에 존재하여 메서드 종료 시 소멸되어 접근이 불가능한 반면, 람다 표현식은 heap 영역에 저장되어 메서드 종료 후에도 존재하여 접근이 가능하다.
2. 1번의 특성에 의해 람다 표현식 외부의 변수를 내부에서 사용하기 위해서는 람다 캡처링이 필요하다.
3. 멀티 스레드에서 람다 캡처링한 값들 간 동시성 문제가 발생하기 때문에 외부의 변수는 불변(immutable)해야 한다.

<br/>

## 참고
- [Lambda와 effecitvely final](https://vagabond95.me/posts/lambda-with-final/)
- [자바의 effectivley final](https://madplay.github.io/post/effectively-final-in-java)
- [JVM 메모리 영역](https://inpa.tistory.com/entry/JAVA-%E2%98%95-JVM-%EB%82%B4%EB%B6%80-%EA%B5%AC%EC%A1%B0-%EB%A9%94%EB%AA%A8%EB%A6%AC-%EC%98%81%EC%97%AD-%EC%8B%AC%ED%99%94%ED%8E%B8#%EC%9E%90%EB%B0%94_%EA%B0%80%EC%83%81_%EB%A8%B8%EC%8B%A0jvm%EC%9D%98_%EA%B5%AC%EC%A1%B0)
- [람다 캡처링](https://catsbi.oopy.io/9b757e48-a756-4469-973e-a06d0f34e7a4)

