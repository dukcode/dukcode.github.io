---
layout: single
title: "[leetcode 7] Reverse Integer - Java"
categories: algorithm
tag: [leetcode]
toc: true
toc_sticky: true
author_profile: false
sidebar:
  nav: "docs"
search: true
---

[[leetcode link]](https://leetcode.com/problems/reverse-integer/)

`x = 1023456789` 일 때 뒤집으면 `res = 9876543201`로 리턴타입인 `int`의 범위를 넘어간다.\
따라서 `res`를 `long`으로 선언하고 뒤집은 후 `int`의 범위 내에 있는지 확인하고 `int`로 명시적 형변환 하여 값을 리턴한다.

## 구현
```java
// [leetcode 0007] Reverse Integer

class Solution
{
    public int reverse(int x)
    {
        long res = 0;
        while (x != 0)
        {
            res *= 10;
            res += x % 10;
            x /= 10;
        }

        if ((long)Integer.MIN_VALUE <= res && res <= (long)Integer.MAX_VALUE)
        {
            return (int)res;
        }

        return 0;
    }
}
```

## 실행결과
<img width="646" alt="image" src="https://user-images.githubusercontent.com/59705184/149063246-36185962-acd8-498c-b114-b89d78d93343.png">
