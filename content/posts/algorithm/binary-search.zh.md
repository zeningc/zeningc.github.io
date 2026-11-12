---
date: '2026-02-28T16:38:35-07:00'
draft: false
title: '面试刷题总结 - 二分查找'
---

# 面试刷题总结 - 二分查找
## 总结
说到二分查找，几乎所有 LeetCode 题目都可以归为以下 3 类：
### 在答案上二分
- 定位目标值的位置，例如给定一个有序数组，找到目标值第一次出现 / 最后一次出现的位置：
- 代码模板：
    ``` java
    int lo = 0;
    int hi = n - 1;
    while (lo <= hi)    {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] <= nums[hi]) // 找第一次出现用 <=；找最后一次出现改用 <
            hi = mid - 1;
        else
            lo = lo + 1;
    }
    ```
- 这是二分查找最基础的形式。有一个在脑中理解它的小技巧：把数组想象成 FFFFXXXXTTTTT 这样的形式，其中 F 是所有小于目标值的元素，X 是等于目标值的元素，T 是大于目标值的元素。然后把 mid 放到其中任意一个元素上，看看应该怎样修改 if 条件，才能让 lo/hi 移动到我们期望的位置。
- 例题：略，这类题太多了

### 结构性二分
- 输入不再有序，而是具有某种结构（比如一个"峰值"，满足 nums[i - 1] < nums[i] > nums[i + 1]），题目通常要求找到任意一个"峰值"。
- 代码模板（这是 162 题，最基础的形式）：
    ``` java
    int lo = 0;
    int hi = n - 1;
    while (lo < hi)    { // 这里不同，因为我们需要给 mid 和 mid + 1 留出空间
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] > nums[mid + 1])
            hi = mid;
        else
            lo = mid + 1;
    }
    return lo; // lo 就是"峰值"
    /*
    我一开始不明白为什么这样能找到峰值，下面是我的思考过程：
    把 lo 和 hi 看作合法的搜索下界/上界（我们相信之前的每一次移动都帮我们排除了不可能是答案的部分 / 保留了可能是答案的部分，就像递归一样）
    如果发现 nums[mid] > nums[mid + 1]，有两种可能：
        1. nums[mid - 1] > nums[mid]，mid 不是候选，但没关系，因为 nums[mid - 1] 成了新的候选；即使 mid - 1 == 0，题目设定里 nums[-1] 也是 -inf
        2. nums[mid - 1] < nums[mid]，mid 就是答案
    否则 nums[mid] < nums[mid + 1]，mid + 1 可能是候选；如果 mid + 1 == n，那 mid + 1 就是答案，否则继续把 mid + 2 作为候选检查，以此类推……
    */
    ```
- 例题：
    - [162. Find Peak Element](https://leetcode.com/problems/find-peak-element/description/)
    - [852. Peak Index in a Mountain Array](https://leetcode.com/problems/peak-index-in-a-mountain-array/description/) - 与 162 相同
    - [1901. Find a Peak Element II](https://leetcode.com/problems/find-a-peak-element-ii/description/)
        - 这是一个二维数组，情况稍有不同。
        - 不过我们可以对行做二分，并在第 `mid` 行取最大值 maxVal。按照同样的思路，可以证明这个策略能帮我们找到答案。
    - [1095. Find in Mountain Array](https://leetcode.com/problems/find-in-mountain-array/description/)
        - 这题结合了`结构性二分`和`在答案上二分`，技巧是先找到峰值，再在两侧分别做二分查找。

### 旋转数组
- 输入在旋转之前是有序的，旋转后数组看起来像 [5,6,7,1,2,3]。这类题的技巧是找出其中一侧的单调性。
- 代码模板（针对基础情形）：
    ``` java
        int lo = 0;
        int hi = n - 1;
        while (lo < hi)    {
            int mid = lo + (hi - lo) / 2;
            if (nums[mid] > nums[hi]) // 判断单调性：如果成立，断点在右侧
                lo = mid + 1;
            else
                hi = mid;
        }

        return nums[lo];
    ```
- 例题：
    - [33. Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array/description/)
        - 这题要定位目标值，需要找出没有断点的那一侧（`mid` 的左边或右边），再决定如何移动 `hi` 或 `lo`。
    - [81. Search in Rotated Sorted Array II](https://leetcode.com/problems/search-in-rotated-sorted-array-ii/description/)
        - 这是去掉了"元素互不相同"保证的 33 题。因此如果遇到 `nums[lo] == nums[mid] && nums[mid] == nums[hi]`，就无法判断断点在哪一侧。这时缩小搜索范围 `lo++;hi--;`，当所有元素都相同时，时间复杂度会退化为 `O(N)`。
    - [153. Find Minimum in Rotated Sorted Array](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/description/)
    - [154. Find Minimum in Rotated Sorted Array II](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array-ii/description/)
        - 去掉"元素互不相同"保证的 153 题，解法是当 `nums[mid] == nums[hi]` 时，用 `hi--` 打破平局
