---
layout: single
title: "[leetcode 83] Remove Duplicates from Sorted List - Java"
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

빈 입력에 대한 고려를 하지 않으면
```
Runtime Error Message:
java.lang.NullPointerException
  at line 9, Solution.deleteDuplicates
  at line 54, __DriverSolution__.__helper__
  at line 84, __Driver__.main
```

위와 같은 에러가 뜰 확률이 높다. 입력이 `null`일 때 처리를 해줘야 한다.

## 구현
```java
// [leetcode 0083] Remove Duplicates from Sorted List

class Solution
{
    public ListNode deleteDuplicates(ListNode head)
    {
        // 입력이 null이면 그대로 head를 return
        if (head == null)
        {
            return head;
        }

        ListNode pivot = head;
        int pivotVal = 0;

        //위처럼 null입력에 대한 처리를 하지 않으면 pivot.next를 참조해 NullPointException이 발생한다
        while (pivot.next != null)
        {
            pivotVal = pivot.val;
            if (pivotVal == pivot.next.val)
            {
                pivot.next = pivot.next.next;
            }
            else
            {
                pivot = pivot.next;
            }
        }

        return head;
    }
}
```
## 실행결과
<img width="655" alt="image" src="https://user-images.githubusercontent.com/59705184/149065596-5b693b7c-cffd-4008-a74a-e6fb6009f690.png">

