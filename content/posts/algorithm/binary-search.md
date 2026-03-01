---
date: '2026-02-28T16:38:35-07:00'
draft: false
title: 'Coding Interview Prep Summary - Binary Search'
---

# Coding Interview Prep Summary - Binary Search
## Summary
When it comes to binary search, nearly all leetcode problems can be divided into 3 types:
### Binary Search on Answer
- locate the position of target, e.g. given a sorted array, find the first occurance / last occurance of target:
- code template:
    ``` java
    int lo = 0;
    int hi = n - 1;
    while (lo <= hi)    {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] <= nums[hi]) // for first occurance, for last occurance use < instead of <=
            hi = mid - 1;
        else
            lo = lo + 1;
    }
    ```
- this is the most fundamental format of binary search, a trick to illustrate it in your mind is to construct an array like FFFFXXXXTTTTT, where Fs are all elements that are less than your target, Xs are the elements that are equal to your target, and Ts are the elements that are larger than your target. Then, place mid on any of these element to see how we could change if statement to make lo/hi move to our expected postition.
- Sample Question: Skip, there are too many of those

### Structural Binary Search
- The input is no longer sorted, instead, it has some structure (like a "peak" where nums[i - 1] < nums[i] > nums[i + 1]), it usually asks you to find any "peak".
- code template (this is for 162, the most basic form):
    ``` java
    int lo = 0;
    int hi = n - 1;
    while (lo < hi)    { // this part is different because we want to have headroom for mid and mid + 1
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] > nums[mid + 1])
            hi = mid;
        else
            lo = mid + 1;
    }
    return lo; // lo is the "peak"
    /*
    I was confused about why this can help us get a peak. here's my thinking process:
    treat lo and hi as a valid search lower/upper bound(we trust that previous move help us filter out what couldn't be the answer / filter what could be the answer, just like a recursion)
    then if we found out that nums[mid] > nums[mid + 1], there are 2 possibility:
        1. nums[mid - 1] > nums[mid], making mid a non-candidate, but it is okay, since we know nums[mid - 1] is the new candidate, even if mid - 1 == 0, the nums[-1] is -inf in the setting
        2. nums[mid - 1] < nums[mid], mid is the answer
    otherwise nums[mid] < nums[mid + 1], mid + 1 could be a candidate, if mid + 1 == n, then mid + 1 is the answer, otherwise we check mid + 2 as candidate, etc...
    */
    ```
- Sample Problems:
    - [162. Find Peak Element](https://leetcode.com/problems/find-peak-element/description/)
    - [852. Peak Index in a Mountain Array](https://leetcode.com/problems/peak-index-in-a-mountain-array/description/) - same as 162
    - [1901. Find a Peak Element II](https://leetcode.com/problems/find-a-peak-element-ii/description/)
        - This is an 2D array, which makes thing a little bit different.
        - But we can do binary search on rows, and pick the maxVal on row `mid`. Following the same thinking process, we can prove that this strategy is going to help us find the answer.
    - [1095. Find in Mountain Array](https://leetcode.com/problems/find-in-mountain-array/description/)
        - This combines `Structural Binary Search` and `Binary Search on Answer`, the trick is to find the peak first, then do binary search on both sides.

### Rotated Array
- The input was sorted before rotation, making the array appear like [5,6,7,1,2,3]. The trick for these kind of problem is to identify the monotonicity on one side.
- code template (it's for the base case)
    ``` java
        int lo = 0;
        int hi = n - 1;
        while (lo < hi)    {
            int mid = lo + (hi - lo) / 2;
            if (nums[mid] > nums[hi]) // identify monotonicity, if this is the case the break point is to the right
                lo = mid + 1;
            else
                hi = mid;
        }

        return nums[lo];
    ```
- Sample Problems:
    - [33. Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array/description/)
        - this is trying to locate a target, we need to find the range without breakpoint(either on the left or right of `mid`), and then decide how to move `hi` or `lo`.
    - [81. Search in Rotated Sorted Array II](https://leetcode.com/problems/search-in-rotated-sorted-array-ii/description/)
        - this is 33 without unique value guarantee, thus if we see `nums[lo] == nums[mid] && nums[mid] == nums[hi]` we won't be able to tell which side has the breakpoint. Thus we shrink the search field `lo++;hi--;`, which will degrade to `O(N)` time complexity when all elements are the same.
    - [153. Find Minimum in Rotated Sorted Array](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/description/)
    - [154. Find Minimum in Rotated Sorted Array II](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array-ii/description/)
        - 153 without unique guarantee, the solution is when `nums[mid] == nums[hi]`, use `hi--` to break tie
        