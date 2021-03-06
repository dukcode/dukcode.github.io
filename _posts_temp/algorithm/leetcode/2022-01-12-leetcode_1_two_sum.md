---
layout: single
title: "[leetcode 1] Two Sum - Java"
categories: algorithm
tag: [leetcode]
toc: true
toc_sticky: true
author_profile: false
sidebar:
  nav: "docs"
search: true
---

[[leetcode link]](https://leetcode.com/problems/two-sum/)

**Brute Force** 방법으로 모든원소를 순회하며 두 원소의 합이 `target`과 같으면 배열을 `return`한다.  
$O(n^2)$ 시간복잡도를 가지게 된다.

## Solution #1
### 구현
```java
// [leetode 0001] Two Sum

class Solution
{
    public int[] twoSum(int[] nums, int target)
    {
        for (int i = 0; i < nums.length - 1; ++i)
        {
            for (int j = i + 1; j < nums.length; ++j)
            {
                if (nums[i] + nums[j] == target)
                {
                    return new int[]{i, j};
                }
            }
        }

        return null;
    }
}
```

### 실행결과
<img width="609" alt="image" src="https://user-images.githubusercontent.com/59705184/149057162-bff347b2-66d5-4c32-88bb-d152d6b4e3f8.png">

약 하위 10%의 속도로 알고리즘을 개선할 필요가 있어 보인다.

---

## Solution #2
### 구현
`find()`에 $O(1)$의 시간복잡도를 가지는 `HashMap`을 사용하여 풀이를 개선시키자
```java
// [leetode 0001] Two Sum
import java.util.HashMap;

class Solution
{
    public int[] twoSum(int[] nums, int target)
    {
        HashMap<Integer, Integer> map = new HashMap<Integer, Integer>();

        for (int i = 0; i < nums.length ; ++i)
        {
            int diff = target - nums[i];

            if (map.containsKey(diff))
            {
                return new int[]{i, map.get(diff)};
            }
            map.put(nums[i], i);
        }

        return null;
    }
}
```
### 실행결과 
<img width="606" alt="image" src="https://user-images.githubusercontent.com/59705184/149061451-c38969c2-a7d0-4753-b727-8a658ca75043.png">
