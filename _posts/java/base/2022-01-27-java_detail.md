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

자바 기본 문법을 공부하며 대략적인 감을 잡았지만, 복습하면서 기억나지 않거나 새로 알게된 부분, 또는 기억할만 한 부분을 기록하는 포스트이다.
## 소스파일 관련
### 클래스 이름과 소스파일의 이름은 같아야 한다.
* 정확히 말하면 소스파일 내에 포함되어 있는 `public class`의 이름과 소스파일의 이름이 같아야 한다.

### 패키지 선언문을 쓰지 않으면 자동적으로 **이름없는 패키지**에 속하게 된다.
* 클래스패스에 해당 루트 폴더를 포함해야 패키지에 불러올 수 있다.

### import문은 패키지명 생략을 위해 붙인다.
* 클래스패스에서 검색되는 클래스를 찾음

### 모든 소스파일에는 묵시적으로 `import java.lang.*`가 선언되어있다.


## 연산자
### `static import`문을 사용하면 클래스의 static멤버를 호출할 때 클래스 이름을 생략할 수 있다.

```java
import static java.lang.Integer.*       // Integer 클래서의 모든 static 메서드
import static java.lang.Math.random;    // Math.random()만, 괄호 안붙임
```

## 분기, 반복문
### switch문의 조건식에는 정수 이외에 문자, 문자열이 가능하다.

### 반복문에 이름을 붙여 중첩반복문을 빠져나갈 수 있다.

```java
Loop : for (int i = 0; i < 9; ++i) {
    for (int j = 0; j < 9; ++j) {
        if (i == 5) {
            break Loop;
        }
    }
}
```
### 리터럴 간의 연산은 컴파일 시에 컴파일러가 계산하여 결과로 대체한다.
* 따라서 아래와 같은 코드에서 명시적 형변환 과정을 거치지 않고 컴파일이 가능하다.

```java
char c1 = 'a';

// char c2 = c1 + 1;    // c1 + 1이 int형이므로 자동 형변환 안됨
char c2 = 'a' + 1;  // 가능
```
### 향상된 for문에서는 배열에 저장된 값을 변경할 수 없다.

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


## 배열
### 배열의 길이는 0일수도있다.

```java
int[] arr = new int[0];
```

### println의 매개변수에 배열을 넣으면 Type@Address가 출력되지만 char배열은 예외이다.

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

## 함수

### return 형이 void인 메서드의 경우 컴파일러가 자동으로 return;을 추가해준다.

### 가변인자(varargs)는 내부적으로 배열을 이용한다.
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

### `main()`의 `args`는 입력된 매개변수가 없을 때, null 대신 크기가 0인 배열이 된다.

## 클래스
### 생성자가 없으면 컴파일러가 기본적으로 기본 생성자를 생성해 컴파일 한다.

### 생성자에서 다른 생성자의 호출은 첫줄에서만 가능하다.

### 멤버변수는 초기화를 하지 않아도 자동적으로 기본값으로 초기화된다.
* 지역변수는 초기화 되지 않는다.

### 초기화 블럭은 클래스가 메모리에 로딩될 때 한번만 수행된다.
* 인스턴스 초기화 블럭은 인스턴스를 생성할 때 마다 수행된다.
* 모든 초기화 블럭은 생성자 호출 전에 수행된다.

### 클래스 변수 초기화 순서
* 시점 : 클래스 로딩 시
* 기본값 -> 명시적 초기화 -> 클래스 초기화 블럭

### 인스턴스 변수 초기화 순서
* 시점 : 각 인스턴스 생성 시
* 기본값 -> 명시적 초기화 -> 인스턴스 초기화 블럭 -> 생성자

### `final`이 붙은 멤버변수는 클래스 초기화 블럭이나 생성자에서 초기화가 가능하다.

* 코드

```java
class Test {
    public static final int STATIC_VALUE;
    public final int INSTANCE_VALUE;

    static {
        STATIC_VALUE = 10;
    }

    public Test() {
        INSTANCE_VALUE = 100;
    }

    public static void main(String[] args) {
        Test t = new Test();
        System.out.println(Test.STATIC_VALUE);
        System.out.println(t.INSTANCE_VALUE);
    }
}
```

* 실행 결과

```sh
10
100
```

## 상속
### 모든 클래스의 조상 클래스는 `Object` 클래스이다.
* 다른 클래스로부터 상속 받지 않는 클래스의 경우 컴파일러가 자동으로 `extends Object`를 추가하여 컴파일한다.

### 메서드 오버라이딩 시에 접근 제어자는 조상 클래스의 메서드 보다 좁은 범위로 변경할 수 없다.

### 메서드 오버라이딩 시에 예외는 조상 클래스의 메서드 보다 많이 선언할 수 없다.

### 조상 클래스에 정의된 static 메서드와 똑같은 이름의 메서드를 자식 클래스에 정의한 것은 오버라이딩이 아니다.
* 각각 클래스에 별개의 static메서드를 정의한 것일 뿐이다.

### `this`와 `super`는 static 메서드에서 사용할 수 없다.
* `this`와 `super`는 그 자체로 인스턴스의 주소를 나타낸다.

### `this()` 또는 `super()`는 모든 클래스의 생성자의 첫줄에 호출해야 한다.
* 호출하지 않는 경우 컴파일러가 자동적으로 조상 클래스 기본 생성자를 추가한다.
* 이 때 조상 클래스의 기본 생성자가 없는 경우 컴파일 에러가 발생한다.

### 생성자의 접근 제어자
* 보통 생성자의 접근 제어자는 클래스의 접근 제어자와 같다.
* 생성자의 접근 제어자를 private로 한다면 클래스 내부에서 인스턴스를 생성해 return하는 방식을 통해 인스턴스의 갯수를 제한할 수 있다.
* 생성자의 접근 제어자를 private로 한다면 인스턴스 생성을 막을 수 있다. (ex : Math 클래스)

### 메서드에 static과 abstract를 함께 사용할 수 없다.
* static메서드는 overriding이 되지 않는다.
* 따라서 static메서드는 몸통이 있는 메서드에서만 사용할 수 있다.

### 클래스에 abstract와 final을 동시에 사용할 수 없다.
* abstract class = 상속을 통해 완성되어야 하는 클래스
* final class = 상속을 할 수 없는 클래스

### abstract 메서드의 접근 제어자가 private일 수 없다.
* abstract메서드는 자손 클래스에서 구현해주어야 하는데 private이면 자손 클래스에서 접근 불가하다.

### 메서드에 private와 final을 같이 사용할 필요는 없다.
* private메서드는 오버라이딩할 수 없기 때문이다.

### 서로 상속관계에 있는 클래스들 끼리만 참조변수의 형변환이 가능하다.
* [업캐스팅] 자손타입 -> 조상타입 : 형변환 생략 가능
* [다운캐스팅] 조상타입 -> 자손타입 : **형변환 생략 불가**

### 형변환은 인스턴스에 아무런 영향을 미치지 못한다.
* 참조변수의 타입을 변환하는 것이지 인스턴스를 변환하는 것은 아니기 때문
* 단지 참조변수의 형변환을 통해 인스턴스에서 사용할 수 있는 멤버의 범위를 조절하는것 뿐이다.

### 다운캐스팅해서 멤버변수를 호출하면 런타임에러가 생성된다.
* 인스턴스가 자손클래스의 인스턴스이고 다운캐스팅으로 인해 자신의 특성을 잃었을때 사용한다.
* 다운캐스팅의 이유는 업캐스팅으로 잃어버린 자손인스턴스의 특성을 복구시키기 위함이다.

### 메서드는 인스턴스를 따라가고, 멤버변수는 참조변수를 따라간다.
* 코드

```java
class Child extends Parent {
    String str = "child String";

    void foo() {
        System.out.println("child foo()");
        System.out.println(str + " in foo()");
    }
}

class Parent {
    String str = "parent String";

    void foo() {
        System.out.println("parent foo()");
        System.out.println(str + " in foo()");
    }

    public static void main(String[] args) {
        Parent p = new Child();
        System.out.println(p.str);
        p.foo();
    }
}
```

* 실행결과

```sh
parent String
child foo()
child string in foo()
```

멤버변수가 조상 클래스와 자손 클래스에 중복으로 정의된 경우, 메서드는 인스턴스를 따라가고, 멤버변수를 참조변수를 따라간다.
**그 이유는 메서드는 오버라이딩이므로 조상, 자손 클래스에 대해서 같은 상대주소를 가지지만, 멤버변수는 오버라이딩 된 것이 아니기 때문이다.** 자손클래스에서 조상클래스의 멤버변수를 접근하려면 super을 사용할 수 있다.


## 인터페이스
### 추상메서드를 포함하지 않은 클래스에도 abstract를 붙여서 추상클래스로 지정할 수 있다.
* 추상클래스로 지정되면 클래스의 인스턴스를 생성할 수 없기 때문에 인스턴스 생성을 막기 위해서 abstract를 붙일 수 있다.

### 추상메서드를 포함한 클래스에 abstract를 붙이지 않으면 컴파일 에러가 발생한다.

### interface도 class와 같이 접근제한자로 (default)나 public을 사용할 수 있다.
### interface의 모든 멤버변수는 `public static final`이며 컴파일러가 자동으로 추가한다.
### interface의 모든 메서드는 `public abstract`이며 컴파일러가 이를 자동으로 추가한다.
### interface는 class와 달리 다중상속이 가능하다.
### interface를 구현하는 class가 메서드 중 일부만 구현한다면 abstract를 붙여 추상클래스로 선언해야한다.
### interface를 구현하고 조상클래스 상속을 동시에 하는 경우, static상수의 충돌이 일어날 수 있는데 이는 클래스의 이름을 붙여 해결 가능하다.
### interface를 구현하고 조상클래스 상속을 동시에 하는 경우, interface에 선언되어 있는 메서드를 조상클래스가 가지고 있을 경우 구현한 것으로 간주한다.
* 코드

```java
interface Talkable {
    public abstract void talk();
}

class Child extends Parent implements Talkable {

}

class Parent {
    public void talk() {
        System.out.println("parent talk");
    }

    public static void main(String[] args) {
        Child c = new Child();
        c.talk();
    }
}
```

* 실행 결과

```sh
parent talk
```

### 다중 상속의 효과를 내고 싶다면 한 클래스는 상속하고 한 클래스는 클래스 내부로 포함시킨 후 interface를 가지고 구현하면 다형적 특성을 이용할 수 있다.
```java
class Watch {
    public void getTIme() {
        // 시간을 print
    }
}

class Computer {
    public void websurfing() {
        // 웹서핑
    }
}

interface InterfaceComputer {
    void websurfing();
}

class AppleWatch extends Watch implements InterfaceComputer {
    Computer c = new Computer();

    public void websurfing() {
        c.websurfing();
    }
}
```

### interface 타입의 참조변수에서 Object에 정의된 메서드들을 호출할 수 있다.
* 모든 객체는 Object클래스에 정의된 메서드를 가지고 있을 것이기 때문에 허용한다.

### JDK1.8부터 interface에 디폴트 메서드와 static메서드를 추가 할 수 있게 되었다.
* 따라서 JDK1.8 이전엔 인터페이스와 관련된 static메서드는 별도의 클래스에 따로 두어야 했다.
* Collection 인터페이스가 static메서드를 가질 수 없었으므로 별도의 Collections 클래스를 두어 static메서드를 두었다.

### interface의 디폴트 메서드를 활용하면 메서드를 추가해도 해당 interface를 구현한 클래스들에 메서드를 구현하지 않아도된다.
* 디폴트 메서드는 메서드 앞에 default를 붙인다.
* 접근제어자는 public이고 생략 가능하다.

### 여러 interface를 구현할 때 interface들의 디폴트 메서드가 충돌한다면 디폴트 메서드를 오버라이딩 해야한다.
### intferface의 디폴트 메서드와 조상클래스의 메서드가 충돌할 때는 디폴트 메서드는 무시된다.

## 내부 클래스
### 내부 클래스는 abstract, final같은 제어자를 사용할 수 있을 뿐만 아니라, 멤버변수들 처럼 private, protected 접근제어자도 사용 가능하다.
### 내부 클래스가 static인 경우에만 static 멤버 변수를 가질 수 있다.
* final static은 상수이므로 static 클래스가 아니어도 허용된다.

### 지역 클래스는 메서드에 정의된 final이 붙은 지역변수를 사용할 수 있다.
* 메서드가 수행을 마쳐 지역변수가 소멸된 시점에 지역클래스의 인스턴스가 소멸된 지역변수를 참조하려는 경우가 발생할 수 있기 때문이다.
* JDK1.8부터 내부지역클래스가 로컬변수를 참조할 때 편의를 위해 자동적으로 final을 붙여준다. **하지만 로컬 변수의 변경이 있다면 컴파일 오류가 발생한다.**

### 내부클래스를 컴파일 했을 때 생성되는 파일명은 **외부클래스명$내부클래스명.class**이다.
* 지역내부클래스는 내부클래스명 앞에 숫자가 붙게 되는데 이는 다른 메서드에 같은 이름의 지역내부클래스가 선언되 있을 수 있기 때문이다.

### 내부클래스에서 외부클래스의 멤버변수에 접근할 때는 `Outer.this.value`와 같은 형태로 접근할 수 있다.

## 예외

### Exception의 자손은 RuntimeException과 그 외의 클래스로 나눌 수 있다.
* 그 외의 클래스를 Exception클래스들 이라고 하자.
* RuntimeException은 프로그래머의 실수에 의해 발생 할 수 있는 예외들이다. (ex. ArrayOutOfBoundsExceptions, NullpointerException, ClassCastException, ArithmeticException)
* Exception클래스들은 주로 사용자들의 동작에 의해 발생하는 경우가 많다. (ex. 없는 파일의 이름 입력, 입력한 데이터 형식의 오류)

### 처리하지 못한 에러는 JVM의 예외처리기(UncaughtExceptionHandler)가 받아서 예외의 원인을 화면에 출력한다.

### try-catch문은 블럭 내에 포함된 문장이 하나뿐이라도 괄호를 생략할 수 없다.

### catch블럭의 괄호 내에 선언된 참조션수의 종류와 생성된 예외클래스의 인스턴스를 instanceof 연산차를 이용해 검사한다.

### 검사에서 일치하는 블럭을 찾으면 다음 블럭은 검사하지 않는다.

### `printStackTrace()`는 예외발생 당시의 호출스택에 있었던 메서드의 정보와 예외메시지를 화면에 출력한다.

### `getMessage()`는 발생한 예외클래스이 인스턴스에 저장된 메시지를 얻을 수 있다.

### JDK1.7부터 '|' 기호를 이용해 catch블록에서 여러 exception을 처리할 수 있다.

```java
try {
    ...
} catch (ExcaptionA | ExceptionB e) {
    e.printStackTrace();
}
```

* '|' 기호로 연결된 예외 클래스가 조상과 자손관계에 있다면 컴파일 에러가 발생한다.
* 두 클래스의 조상을 매개변수로 받으면 되기 때문에 불필요한 코드를 제거하라는 의미이다.
* e로 조상 예외 클래스에 선언된 멤버만 사용할 수 있다.
* instanceof 연산자로 e가 어떤 예외의 인스턴스인지 알아내어 각각 처리가 가능하지만, 이렇게 하면 catch문을 합치는 이유가 없다.

### RuntimeException은 처리해주지 않아도 컴파일이 된다.
* 원칙적으로 모든 Exception은 처리를 해주어야 한다. 처리해주지 않으면 컴파일 에러가 발생한다.
* Exception이 발생하면 자손엔 RuntimeException 이외의 클래스들이 포함되어 있기 때문에 항상 처리가 필요하다. (안하면 컴파일 에러)
* RuntioneException을 던지고 처리하지 않는다면 컴파일은 된다.
    * RuntimeException은 프로그래머의 실수와 관련 있는데, 다음 코드와 같이 매번 Exception을 처리해주어야 하면 불편할 것이다.
    ```java
    try {
        int[] arr = new int[10];

        ...

        // arr이 null일 수 있음, 0이 array bound를 벗어날 수 있음.
        System.out.println(arr[0]);
    } catch (ArrayIndexOutOfBoundsException ae) {
        ...
    } catch (NullpointerException ne) {
        ...
    }
    ```

### 메서드 내부에서 Exception을 던진다면 메서드 선언부에 throws로 예외를 선언해주어야 컴파일 에러가 나지 않는다.

### 메서드 선언부의 throws에는 보통 반드시 처리해주어야 하는 예외만 선언한다.
* 일레로 Object의 클래스의 wait()메서드는 InterruptedException과 IllegalMonitorStateException을 실제로 던지지만 첫번째 예외는 Exception의 직계자손이고, 후자는 Runtime의 직계자손이라 메서드 선언부에서는 InterruptedException만 던진다.

### try-catch-finally 블럭에서 try 블럭에 return문이 있더라고 return 실행 전에 finally블럭에 있는 코드가 실행되고 return된다.

### catch블럭에서 예외를 처리하고 다시 같은 예외를 던짐으로써 양쪽에서 나눠서 예외를 처리할 수 있다. 이를 예외 되던지기 라고 한다.

### 일반적으로 return값이 void가 아닌 함수는 try문과 catch문 안에 return이 필요하지만, 예외 되던지기를 할 때 catch문에서 return문 대신 예외를 던진다.
