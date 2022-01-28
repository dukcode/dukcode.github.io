---
layout: single
title: "[JAVA] 자바 문법 디테일(작성중)"
categories: java
tag: [java]
toc: true
toc_sticky: true
author_profile: false
sidebar:
  nav: "docs"
search: true
---

## 클래스 이름과 소스파일의 이름은 같아야 한다.
* 정확히 말하면 소스파일 내에 포함되어 있는 `public class`의 이름과 소스파일의 이름이 같아야 한다.

## 리터럴 간의 연산은 컴파일 시에 컴파일러가 계산하여 결과로 대체한다.
* 따라서 아래와 같은 코드에서 명시적 형변환 과정을 거치지 않고 컴파일이 가능하다.
```java
char c1 = 'a';

// char c2 = c1 + 1;    // c1 + 1이 int형이므로 자동 형변환 안됨
char c2 = 'a' + 1;  // 가능
```

## switch문의 조건식에는 정수 이외에 문자, 문자열이 가능하다.

## 반복문에 이름을 붙여 중첩반복문을 빠져나갈 수 있다.
```java
Loop : for (int i = 0; i < 9; ++i) {
    for (int j = 0; j < 9; ++j) {
        if (i == 5) {
            break Loop;
        }
    }
}
```
## 배열의 길이는 0일수도있다.
```java
int[] arr = new int[0];
```

## println의 매개변수에 배열을 넣으면 Type@Address가 출력되지만 char배열은 예외이다.
```java
int[] arr = {1, 2, 3, 4, 5};
char[] chArr = {'a', 'b', 'c', 'd', 'e'};

System.out.println(arr);
System.out.println(chArr);
```
* 실행 결과
```sh
[I@14318bb
abcde
```

## `main()`의 `args`는 입력된 매개변수가 없을 때, null 대신 크기가 0인 배열이 된다.

## 향상된 for문에서는 배열에 저장된 값을 변경할 수 없다.
```java
int[] arr = {1, 2, 3, 4, 5}
for (int num : arr) {
    num *= 2;
}
```
위의 코드를 풀어 쓰면
```java
int[] arr = {1, 2, 3, 4, 5}
for (int i = 0; i < arr.length; ++i) {
    int num = i;
    num *= 2;
}
```
위와 같기 때문이다. (임시로 복사해서 쓴다.)

## return 형이 void인 메서드의 경우 컴파일러가 자동으로 return;을 추가해준다.

## 가변인자(varargs)는 내부적으로 배열을 이용한다.
* 따라서 인자를 생략하고 싶을 때 `null`이나 길이가 0인 배열을 인자로 지정해야 한다.
```java
String concatenate(String... str) {
    ...
}

// concatenate();               // 에러
concatenate(new String[0]);     // 가능
concatenate(null);              // 가능

```
* 또한 아래와 같은 경우 인자를 구별할 수 없어 오버로딩에 오류가 생긴다.
```java
String concatenate(String... str) {
    ...
}

String concatenate(String delim, String str) {
    ...
}
```

## 생성자가 없으면 컴파일러가 기본적으로 기본 생성자를 생성해 컴파일 한다.

## 생성자에서 다른 생성자의 호출은 첫줄에서만 가능하다.

## 멤버변수는 초기화를 하지 않아도 자동적으로 기본값으로 초기화된다.
* 지역변수는 초기화 되지 않는다.

## 초기화 블럭은 클래스가 메모리에 로딩될 때 한번만 수행된다.
* 인스턴스 초기화 블럭은 인스턴스를 생성할 때 마다 수행된다.
* 모든 초기화 블럭은 생성자 호출 전에 수행된다.

## 클래스 변수 초기화 순서
* 시점 : 클래스 로딩 시
* 기본값 -> 명시적 초기화 -> 클래스 초기화 블럭

## 인스턴스 변수 초기화 순서
* 시점 : 각 인스턴스 생성 시
* 기본값 -> 명시적 초기화 -> 인스턴스 초기화 블럭 -> 생성자

## 모든 클래스의 조상 클래스는 `Object` 클래스이다.
* 다른 클래스로부터 상속 받지 않는 클래스의 경우 컴파일러가 자동으로 `**extends** Object`를 추가하여 컴파일한다.

## 메서드 오버라이딩 시에 접근 제어자는 조상 클래스의 메서드 보다 좁은 범위로 변경할 수 없다.

## 메서드 오버라이딩 시에 예외는 조상 클래스의 메서드 보다 많이 선언할 수 없다.

## 조상 클래스에 정의된 static 메서드와 똑같은 이름의 메서드를 자식 클래스에 정의한 것은 오버라이딩이 아니다.
* 각각 클래스에 별개의 static메서드를 정의한 것일 뿐이다.

## `this`와 `super`는 static 메서드에서 사용할 수 없다.
* `this`와 `super`는 그 자체로 인스턴스의 주소를 나타낸다.

## 모든 클래스의 생성자의 첫줄에 `this()` 또는 `super()`을 호출해야 한다.
* 호출하지 않는 경우 컴파일러가 자동적으로 조상 클래스 기본 생성자를 추가한다.
* 이 때 조상 클래스의 기본 생성자가 없는 경우 컴파일 에러가 발생한다.


## 자주 쓰이는 함수
