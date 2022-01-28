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

## return 형이 void인 메서드의 경우 컴파일러가 자동으로 return; 추가해준다.

## 자주 쓰이는 함수
