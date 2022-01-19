---
layout: single
title: "[JAVA] clone()의 내부 구현 추측"
categories: java
tag: [java, markInterface]
toc: true
toc_sticky: true
author_profile: false
sidebar:
  nav: "docs"
search: true
---

java의 모든 클래스의 최상위 부모 클래스는 `Object` 클래스 이다. 따라서 모든 클래스에서 `Object` 클래스에 구현되어 있는 메서드를 사용할 수 있다. 하지만 공부를 하다 보면 `Object` 클래스의 `clone()` 메서드는 오버라이딩 시에 `Child Class`가 `Cloneable` 인터페이스를 구현해야 해야 한다고 한다. 그 이유가 궁금해서 여러 정황에 따라 추측 해보려 한다.


## clone() 메서드

`Object` 클래스의 `clone()` 메서드 소스를 확인해 보자.

```java
    protected native Object clone() throws CloneNotSupportedException;
```
위와 같이 [native](https://fors.tistory.com/80)로 구현되어 있어 내부가 어떻게 생겨 먹었는지 알수가 없다. 다만 `CloneNotSupportedException` 예외를 던지는 함수임을 알 수 있다. 오버라이딩 시에 `Cloneable`인터페이스를 구현하지 않으면 해당 예외를 던진다.

## Cloneable 인터페이스

`Cloneable`의 인터페이스의 소스를 보면

```java
public interface Cloneable {
}
```

위와 같이 내부에 아무것도 구현되있지 않다. 그저 [마크인터페이스](https://marshallslee.tistory.com/entry/자바-마커-인터페이스란)이다.

## `clone()`의 내부 구현 추측?

### 추측의 전제

* class 내의 모든 멤버 변수(얼마나 있을지 모를)를 복사하려면 마지막 멤버변수를 가르키는 pointer가 필요하다.
* pointer는 java에서 지원하지 않기 때문에 native로 구현된 `super.clone()`을 호출해야 모든 멤버 변수를 복사할 수 있다.
* **따라서 `clone()` 메서드를 오버로딩 할 때는 항상 `super.clone()`을 호출한다.**
* 결과적으로`super.clone()`을 사용할 것이 아니라면 굳이 clone()을 오버라이딩 할 필요가 없다.

위의 전제를 가지고 인터페이스를 구현하지 않으면 오버로딩(super의 메서드를 호출하는)이 불가능한 코드를 짜보았다.

### 구현 

* 마크 인터페이스
```java
interface MarkInterface
{}
```

* 부모 클래스
```java
class ParentClass
{
    public void foo() throws Exception
    {
        if (this instanceof MarkInterface == false)
        {
            throw new Exception("does not implement MarkInterface\n");
        }

        System.out.println("Do Shallow Copy");
    }
}
```
**this의 type이 마크인터페이스를 구현하지 않으면 예외를 던진다.**

* 마크 인터페이스를 구현한 자식 클래스
```java
class ChildClass1 extends ParentClass implements MarkInterface
{
    public void foo()
    {
        try
        {
            super.foo();
        }
        catch (Exception e)
        {
            System.out.print(e);
        }
    }
}
```

* 마크 인터페이스를 구현하지 않은 자식 클래스
```java
class ChildClass2 extends ParentClass
{
    public void foo()
    {
        try
        {
            super.foo();
        }
        catch (Exception e)
        {
            System.out.print(e);
        }
    }
}
```

* 실행
```java
class Main
{
    public static void main(String args[])
    {
        ChildClass1 c1 = new ChildClass1();
        ChildClass2 c2 = new ChildClass2();

        c1.foo();
        c2.foo();
    }
}
```

* 실행 결과
```sh
Do Shallow Copy
java.lang.Exception: does not implement MarkInterface
```
