---
layout: single
title: "[JAVA] String 클래스"
categories: java
tag: [java, String]
toc: true
toc_sticky: true
author_profile: false
sidebar:
  nav: "docs"
search: true
---
> 코딩테스트나, 프로젝트를 준비하며 사용해본 경험이 있는 생성자와 메서드를 정리할 예정이다.

## 생성자
...작성예정
## 메서드
### `startsWith()`, `endsWith()` 메서드
`startsWith()`, `endsWith()`의 내부구현을 보면 다음과 같다.
```java
    public boolean startsWith(String prefix) {
        return startsWith(prefix, 0);
    }

    public boolean endsWith(String suffix) {
        return startsWith(suffix, length() - suffix.length());
    }
```

위 2개의 메서드는 해당 `String`이 특정 prefix로 시작하는지, 또는 특정 suffix로 끝나는지 확인해 `boolean` type으로 `return`하는 메서드이다. 내부적으로 같은 메서드를 사용하는데 해당 메서드의 내부구현은 아래와 같다. (인코딩 관련 내용을 삭제하고(`coder == LATIN1`인 경우에만 남기고) 임의로 간략화 하였다.)

```java
    public boolean startsWith(String prefix, int toffset) {
        // prefix가 시작할 index가 유효한 index인지 확인
        if (toffset < 0 || toffset > length() - prefix.length()) {
            return false;
        }

        byte ta[] = value;
        byte pa[] = prefix.value;

        int po = 0;
        int pc = pa.length;
        int to = toffset;

        while (po < pc) {
            if (ta[to++] != pa[po++]) {
                return false;
            }
        }

        return true;
    }
```


### `trim()` 메서드
`trim()`메서드는 `String`의 앞과 뒤에서 공백문자를 자른 문자열을 `return`한다. 내부구현을 보면 다음과 같다.

```java
    public String trim() {
        String ret = isLatin1() ? StringLatin1.trim(value)
                                : StringUTF16.trim(value);
        return ret == null ? this : ret;
    }
```

`Latin1`형식의 인코딩일 경우 `StringLatin1.trim()`을 실행하고 내부구현은 다음과 같다.

```java
    public static String trim(byte[] value) {
        int len = value.length;
        int st = 0;
        
        while ((st < len) && ((value[st] & 0xff) <= ' ')) {     // 첫 공백이 아닌 문자까지 st 옮기기
            st++;
        }
        
        while ((st < len) && ((value[len - 1] & 0xff) <= ' ')) {// 맨 마지막 공백이 아닌 문자 뒤로 len 옮기기
            len--;
        }

        // 새로운 String return
        return ((st > 0) || (len < value.length)) ?
            newString(value, st, len - st) : null;
    }
```
* 스페이스 이전의 문자는 제어문자 및 whitespace이기 때문에 공백문자 이하로 끊어준다. 아스키코드표는 [Link](https://cs.stanford.edu/people/miles/iso8859.html)에서 확인할 수 있다.

## 작성중...
