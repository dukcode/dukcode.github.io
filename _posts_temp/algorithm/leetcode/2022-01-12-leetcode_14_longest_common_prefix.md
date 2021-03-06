---
layout: single
title: "[leetcode 14] Longest Common Prefix - Java"
categories: algorithm
tag: [leetcode]
toc: true
toc_sticky: true
author_profile: false
sidebar:
  nav: "docs"
search: true
---

[[leetcode link]](https://leetcode.com/problems/longest-common-prefix)

1. **짧은 element**가 기준이 되어야 하므로 길이가 짧은 순서대로 정렬해준다. : $O(lgn)$
2. `strs[0]`에 가장 짧은 element가 오게 되고 `pivot`으로 삼는다.
3. 나머지 `String`들과 pivot의 글자를 하나씩 비교한다. : $O(mn)$

따라서 시간복잡도는 $O(lgn + mn) = O(mn)$이 되게 된다.

## 구현
```java
// [leetcode 0014] Longest Common Prefix
import java.util.Arrays;

class Solution
{
    public String longestCommonPrefix(String[] strs)
    {
        // 빈 입력에 대한 처리
        if (strs.length == 0)
        {
            return new String("");
        }

        // 길이가 짧은 순으로 정렬
        Arrays.sort(strs, new Comparator<String>()
        {
            @Override
            public int compare(String rhs, String lhs)
            {
                return rhs.length() - lhs.length();
            }
        });

        String pivot = strs[0];

        // pivot의 글자들과 하나씩 비교
        for (int i = 0; i < pivot.length(); ++i)
        {
            for (int j = 1; j < strs.length; ++j)
            {
                if (pivot.charAt(i) != strs[j].charAt(i))
                {
                    return pivot.substring(0, i);
                }
            }
        }

        return pivot;
    }
}
```

## 실행결과
![image](https://user-images.githubusercontent.com/59705184/149078148-fc40dd06-5ad7-4765-8632-5dc704436313.png)

