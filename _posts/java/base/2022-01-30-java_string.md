---
layout: single
title: "[JAVA] String 클래스 Method"
categories: java
tag: [java, String]
toc: true
toc_sticky: true
author_profile: false
sidebar:
  nav: "docs"
search: true
---
> 코딩테스트나, 프로젝트를 준비하며 사용해본 경험이 있는 **생성자와 메서드**의 **내부 구현을 뜯어보며**(~~웬만하면~~) 정리할 예정이다.

# 생성자
```java
public String(char value[])
```
...작성예정
# 메서드

## `startsWith()`, `endsWith()` 메서드
`startsWith()`, `endsWith()`의 내부구현을 보면 다음과 같다.
<details>
<summary>내용 보기</summary>
<div markdown="1">       

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



</div>
</details>

## `trim()` 메서드
`trim()`메서드는 `String`의 앞과 뒤에서 공백문자를 자른 문자열을 `return`한다. 내부구현을 보면 다음과 같다.

<details>
<summary>내용 보기</summary>
<div markdown="1">       
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

        // 첫 공백이 아닌 문자까지 st 옮기기
        while ((st < len) && ((value[st] & 0xff) <= ' ')) {
            st++;
        }

        // 맨 마지막 공백이 아닌 문자 뒤로 len 옮기기
        while ((st < len) && ((value[len - 1] & 0xff) <= ' ')) {
            len--;
        }

        // 새로운 String return
        return ((st > 0) || (len < value.length)) ?
            newString(value, st, len - st) : null;
    }
```
* 내부적으로 새로운 `String`을 생성해 `return`함을 알 수 있다.
* 스페이스 이전의 문자는 제어문자 및 whitespace이기 때문에 공백문자 이하로 끊어준다. 아스키코드표는 [Link](https://cs.stanford.edu/people/miles/iso8859.html)에서 확인할 수 있다.

</div>
</details>

## `replace()` 메서드
내부적으로 새로운 `String`을 생성해 `return`한다.

1. `public String replace(CharSequence target, CharSequence replacement)`

<details>
<summary>내용 보기</summary>
<div markdown="1">       
문자열 중의 문자(target)을 새로운 문자열(replacement)로 모두 바꾼 문자열을 `return`한다.

* `target`의 길이가 1일 때
    * `replacement`의 길이가 1이라면 `replace(char oldChar, char newChar)`을 이용한다.
    ```java
            // replace(char oldChar, char newChar) 이용
            if (trgtLen == 1 && replLen == 1) {
                return replace(trgtStr.charAt(0), replStr.charAt(0));
            }
    ```
    * `replacement`의 길이가 1이 아니라면 return String의 총 길이를 계산하고 새로운 배열을 만들어 채운 후 `String(byte[] value, byte coder)` 생성자로 새로운 `String`을 `return`한다.
    ```java
            // StringLatin1 내부 replace 메서드 : 생성될 배열 길이 계산, 배열생성 후 String return
            String ret = (thisIsLatin1 && trgtIsLatin1 && replIsLatin1)
                    ? StringLatin1.replace(value, thisLen,
                                           trgtStr.value, trgtLen,
                                           replStr.value, replLen)
                    : StringUTF16.replace(value, thisLen, thisIsLatin1,
                                          trgtStr.value, trgtLen, trgtIsLatin1,
                                          replStr.value, replLen, replIsLatin1);
    ```
* `target`의 길이가 0일 때
    * `StringBuilder`를 이용해 문자 사이사이에 `replacement`를 삽입해 `return`한다.
    ```java
        } else { // trgtLen == 0
            int resultLen;
            try {
                // 생성될 길이 계산
                resultLen = Math.addExact(thisLen, Math.multiplyExact(
                        Math.addExact(thisLen, 1), replLen));
            } catch (ArithmeticException ignored) {
                throw new OutOfMemoryError("Required length exceeds implementation limit");
            }

            // StringBuilder이용 append
            StringBuilder sb = new StringBuilder(resultLen);
            sb.append(replStr);
            for (int i = 0; i < thisLen; ++i) {
                sb.append(charAt(i)).append(replStr);
            }
            return sb.toString();
        }
    ```
</div>
</details>

2. `public String replace(char oldChar, char newChar)`
<details>
<summary>내용 보기</summary>
<div markdown="1">       
내부구현은 아래와 같다.
```java
    public String replace(char oldChar, char newChar) {
        if (oldChar != newChar) {
            String ret = isLatin1() ? StringLatin1.replace(value, oldChar, newChar)
                                    : StringUTF16.replace(value, oldChar, newChar);
            if (ret != null) {
                return ret;
            }
        }
        return this;
    }
```
`StringLatin1.replace()`를 살펴보면 아래와 같이 새 배열을 생성하고 String을 return함을 알 수 있다.
```java
    public static String replace(byte[] value, char oldChar, char newChar) {
        // oldChar가 Latin1 형식인지 검사
        if (canEncode(oldChar)) {
            int len = value.length;
            int i = -1;
            while (++i < len) {
                // oldChar와 일치하는 index찾기
                if (value[i] == (byte)oldChar) {
                    break;
                }
            }
            if (i < len) {
                // newChar가 Latin1 형식인지 검사
                if (canEncode(newChar)) {
                    // 초기화 생략한 배열 생성
                    byte[] buf = StringConcatHelper.newArray(len);
                    // 복사
                    for (int j = 0; j < i; j++) {    // TBD arraycopy?
                        buf[j] = value[j];
                    }
                    // 교체
                    while (i < len) {
                        byte c = value[i];
                        buf[i] = (c == (byte)oldChar) ? (byte)newChar : c;
                        i++;
                    }
                    return new String(buf, LATIN1);
                } else {
                    // ... 생략
                }
            }
        }
        return null; // for string to return this;
    }
```

</div>
</details>

## indexOf() 메서드
주어진 문자, 문자열이 존재하는지 확인하고 index를 반환한다. **찾지 못하면 -1을 반환한다.**
<details>
<summary>내용 보기</summary>
<div markdown="1">       

```java
    public int indexOf(int ch)
    public int indexOf(int ch, int fromIndex)
    public int indexOf(String str)
    public int indexOf(String str, int fromIndex)
```
위의 모든 메서드들은 모두 하나의 메서드를 실행시킨다.

```java
    public static int indexOf(byte[] value, int valueCount, byte[] str, int strCount, int fromIndex) {
        byte first = str[0];
        int max = (valueCount - strCount);
        for (int i = fromIndex; i <= max; i++) {
            // str의 첫번째 문자와 일치하는 value의 인덱스를 찾는다
            if (value[i] != first) {
                while (++i <= max && value[i] != first);
            }
            // 뒷 부분이 맞는지 확인한다.
            if (i <= max) {
                int j = i + 1;
                int end = j + strCount - 1;
                for (int k = 1; j < end && value[j] == str[k]; j++, k++);
                if (j == end) {
                    // Found whole string.
                    return i;
                }
            }
        }
        // 찾지 못하면 -1 return
        return -1;
    }
```

</div>
</details>
## `join()` 메서드
...작성예정
## `getBytes()` 메서드
...작성예정
## `valueOf()` 메서드

## 작성중...
