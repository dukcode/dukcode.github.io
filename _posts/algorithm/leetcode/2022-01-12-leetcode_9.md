---
layout: single
title: "[leetcode 9] Palindrome Number - Java"
categories: algorithm
tag: [leetcode]
toc: true
toc_sticky: true
author_profile: false
sidebar:
  nav: "docs"
search: true
---

[[leetcode link]](hhttps://leetcode.com/problems/palindrome-number)

java의 `toString()`메서드를 사용하여 주어진 숫자를 `String`화 시키고 양 끝을 `for loop`를 통해 비교한다.\
길이가 홀수일 때 가운데 문자는 비교할 필요가 없으므로 `length / 2`까지만 비교해준다.  

음수 입력은 항상 `false`로 처리해준다.

* 시간복잡도 : $O(n)$
* 공간복잡도 : $O(n)$
    * input num의 길이만큼 String의 공간이 필요하다.

## 구현
```java
// [leetcode 0009] Palindrome Number

class Solution
{
    public boolean isPalindrome(int x)
    {
        if (x < 0)
        {
            return false;
        }
        
        String num = new String(Integer.toString(x));
        int length = num.length();
        int half = length / 2;
        for (int i = 0; i <= half; ++i)
        {
            if (num.charAt(i) != num.charAt(length - 1 - i))
            {
                return false;
            }
        }

        return true;
    }
}
```
## 실행결과
![image](https://user-images.githubusercontent.com/59705184/149073037-6d42d50f-c90c-4663-a628-d08ed965f3d0.png)
