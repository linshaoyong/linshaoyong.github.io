---
layout: post
title: "Binary Search 学习笔记"
date: 2021-04-24 22:12:50 +0800
tags: [binary search, leetcode, algorithm]
---

Binary Search 是一个比较常见的算法，理解起来还算简单，但是有时候实现不太容易。教科书上讲到 Binary Search 时，一般用有序数组查找元素为例子。

```python
def binary_search(array, target):
    left = 0
    right = len(array) - 1
    while left <= right:
        mid = left + (right - left) // 2
        if array[mid] == target:
            return mid
        elif array[mid] > target:
            right = mid - 1
        else:
            left = mid + 1
    return -1
```

LeetCode 上有挺多 Binary Search 的题目，其中 Hard 难度也不少，难点往往在于如何判断出可以用 Binary Search 来解决。事实上能用 Binary Search 求解的问题，都是有共性的。

以猜0到100以内的数为例，

- 答案在某个范围之内， 0 <= answer <= 100
- 中间值可能就是答案，如果不是，有一半的范围可以排除。比如 answer < 50，那么 [50, 100] 可以排除掉，答案在 [0, 50) 之间
- 每次范围折半，直到找到答案

我们以 Leetcode 上一道题目为例，[1011. Capacity To Ship Packages Within D Days](https://leetcode.com/problems/capacity-to-ship-packages-within-d-days/)

> A conveyor belt has packages that must be shipped from one port to another within `days` days.
>
> The `ith` package on the conveyor belt has a weight of `weights[i]`. Each day, we load the ship with packages on the conveyor belt (in the order given by `weights`). We may not load more weight than the maximum weight capacity of the ship.
>
> Return the least weight capacity of the ship that will result in all the packages on the conveyor belt being shipped within `days` days.

有一组货物 weights 按顺序运输，要在 days 天内，问船的最小容量是多少才能完成任务。

- 首先容量至少要等于 weights 的最大值，不然货物无法装上船。其次，容量最大为所有 weights 总和，就可以 1 次完成运输。 max(weights) <= answer <= sum(weights)
- 取中间值，如果该值满足需求，那么比该值大的数都可以完成运输任务。由于要取最小值，那么右半部分可以排除。相反，就排除左半部分
- 每次范围折半，直到找到答案

```rust
struct Solution;

impl Solution {
    pub fn ship_within_days(weights: Vec<i32>, days: i32) -> i32 {
        let mut low = weights.iter().max().unwrap().clone();
        let mut high: i32 = weights.iter().sum();

        while low < high {
            let mid = (low + high) / 2;
            let mut cap = 0;
            let mut need_days = 1;

            // 当容量为mid，计算出所需天数 need_days
            for weight in weights.iter() {
                if (cap + weight) > mid {
                    cap = 0;
                    need_days += 1;
                }
                cap += weight;
            }

            // 如果所需天数超出，说明mid不够大
            if need_days > days {
                low = mid + 1;
            } else {
                // 否则，说明mid太大
                high = mid;
            }
        }
        low
    }
}

#[test]
fn test_ship_within_days() {
    assert_eq!(
        15,
        Solution::ship_within_days(vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10], 5)
    );
    assert_eq!(6, Solution::ship_within_days(vec![3, 2, 2, 4, 1, 4], 3));
    assert_eq!(3, Solution::ship_within_days(vec![1, 2, 3, 1, 1], 4));
}
```

再看一题，[1760. Minimum Limit of Balls in a Bag](https://leetcode.com/problems/minimum-limit-of-balls-in-a-bag/)

> You are given an integer array `nums` where the `ith` bag contains `nums[i]` balls. You are also given an integer `maxOperations`.
>
> You can perform the following operation at most `maxOperations` times:
>
> - Take any bag of balls and divide it into two new bags with a positive number of balls.
>   - For example, a bag of `5` balls can become two new bags of `1` and `4` balls, or two new bags of `2` and `3` balls.
>
> Your penalty is the **maximum** number of balls in a bag. You want to **minimize** your penalty after the operations.
>
> Return *the minimum possible penalty after performing the operations*.

用一个数组 nums 表示 len(nums) 袋球，nums[i] 代表第 ith 袋球的数量。每次操作可以把某袋球分成两袋，目标是最小化袋子的最大球数，问 maxOperations 次操作后，这个最小值是多少。

- 如果可以无限次操作，那么最终每个袋子都只有1个球，1是可能的最小值。如果不操作，那么初始的最大袋子的球数，就是可能的最大值。1 <= answer <= max(nums)
- 取中间值，如果 maxOperations 次操作以内，可以得到该结果，说明还可能再减小结果值，那么右半部分可以排除。相反，就排除左半部分
- 每次范围折半，直到找到答案

```rust
struct Solution;

impl Solution {
    pub fn minimum_size(nums: Vec<i32>, max_operations: i32) -> i32 {
        let mut left = 1;
        let mut right = *nums.iter().max().unwrap();
        while left < right {
            let mid = left + (right - left) / 2;
            let mut operations = 0;
            // 计算出要多少次operations才能得到mid
            for num in &nums {
                operations += (num - 1) / mid;
            }
            // 如果operations超过允许的最大次数，说明mid太小，无法把所有袋子都分到那么小袋
            if operations > max_operations {
                left = mid + 1;
            } else {
                right = mid;
            }
        }
        left
    }
}

#[test]
fn test_minimum_size() {
    assert_eq!(3, Solution::minimum_size(vec![9], 2));
    assert_eq!(2, Solution::minimum_size(vec![2, 4, 8, 2], 4));
    assert_eq!(7, Solution::minimum_size(vec![7, 17], 2));
}
```

LeetCode 讨论区有一篇非常好的文章，分析了 Binary Search 的解法模版，我也是在这篇文章的启发下，完成了一些题目。强烈建议阅读，[Powerful Ultimate Binary Search Template. Solved many problems](https://leetcode.com/discuss/study-guide/786126/Python-Powerful-Ultimate-Binary-Search-Template.-Solved-many-problems)。
