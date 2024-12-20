---
tags:
---

자바에서는 int, double, long, boolean과 같은 기본(Primitive) 타입이 존재한다.   
이런 기본타입을 객체로 다루기 위해서 각각을 클래스로 감싼  Wrapper 클래스라고 한다.

## 변환 테이블

| 기본(Primitive) 타입 | 래퍼(Wrapper) 클래스 |
| ---------------- | ---------------- |
| char             | Character        |
| int              | Integer          |
| float            | Float            |
| long             | Long             |
| boolean          | Boolean          |
아래와 같이 박싱과 언박싱을 통해서 인스턴스 및 객체로 변환을 할 수 있다.
```Java
// 박싱
Integer num = new Integer(21);

// 언박싱
int n = num.intValue();
```
하지만 현재에는 JDK에서 자동으로 처리해 주므로 불필요한 과정이다.
```Java
Integer num = 21;
int n = num;
```

## Wrapper 클래스를 사용하는 이유
- 래퍼 클래스는 기본 데이터 타입인 Object로 변환하여서 객체 지향적으로 활용할 수 있다.
- `java.util` 패키지의 일부 크래스나 메서드는 객체만 처리할 수 있다.
	- 기본 데이터 타입은 객체가 아니므로 래퍼 클래스를 사용해서 해당 API와 호환되도록 해야 한다.
- Collection(List, Set, Map...) 프레임워크의 데이터 구조는 기본 타입은 저장할 수 없고 객체만 저장할 수 있다.
- `Integer.parseInt(String)`과 같은 유용한 메서드를 제공한다.