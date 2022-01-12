---
layout: single
title: "[leetcode 141] Linked List Cycle - Java"
categories: algorithm
tag: [leetcode]
toc: true
toc_sticky: true
author_profile: false
sidebar:
  nav: "docs"
search: true
---

[[leetcode link]](https://leetcode.com/problems/linked-list-cycle)

1. 처음엔 `[key : int val, value : boolean isVisited]`로 구성되는 `HashMap`을 구성해 문제를 해결하려 했지만 테스트케이스에 같은 `val`을 가진 `ListNode`들이 허용되어 이 방법은 사용하기 어렵다고 생각했다.
2. 따라서 `walker`와 `runner` 방식을 사용했다. 한 `loop`에 두칸 씩 가는 `runner`와 한 `loop`에 한칸 씩 까는 `walker`가 만나면 `true`를 `return`하는 방식이다.
3. 여기서 고려할 점은 `runner`가 한 칸씩 갈 때마다 `null`인지 확인해야 한다는 것이다. (`runner`가 `null`이면 Cycle을 가진 `Linked List`가 아니다)
4. 따라서 이것을 고려하여 `while`문을 짜주어야 한다.

## 구현
```java
// [leetcode 0141] Linked List Cycle

class Solution
{
    public boolean hasCycle(ListNode head)
    {
        ListNode walker = head;
        ListNode runner = head;

        while (runner != null)
        {
            runner = runner.next;

            // runner가 한 칸 갈때마다 null인지 확인
            if (runner == null)
            {
                return false;
            }

            if (runner == walker)
            {
                return true;
            }

            // runner가 한 칸 가고 나서 while()문 안의 조건문 실행 (null인지 확인)
            runner = runner.next;
            walker = walker.next;
        }

        return false;
    }
}
```

## 실행결과
<img width="656" alt="image" src="https://user-images.githubusercontent.com/59705184/149082580-00eb4faa-b151-4864-8675-1ad132475179.png">

