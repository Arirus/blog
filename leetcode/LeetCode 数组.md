# 数组

## 11.盛最多水的容器

### 我的思路
双指针思路

```java
fun maxArea(height: IntArray): Int {
    if (height.size <= 1) return 0;
    var head = 0;
    var tail = height.size-1

    var max = 0
    while(head!=tail){
        max =Integer.max(Integer.min(height[head],height[tail])*(tail-head),max)
        if(height[head]<height[tail]){
            head++
        }else
            tail--
    }

    return max;
}
```

## 26. 删除排序数组中的重复项
双指针思路
```java
fun removeDuplicates(nums: IntArray): Int {
    if (nums.size <= 1) return nums.size

    var head = 0

    for (index in 1 until nums.size){
        if (nums[index]!=nums[head]){
            head++;
            nums[head] = nums[index]
        }
    }

    return head+1
}
```

## 加一

给定一个由整数组成的非空数组所表示的非负整数，在该数的基础上加一。

最高位数字存放在数组的首位， 数组中每个元素只存储单个数字。

你可以假设除了整数 0 之外，这个整数不会以零开头。

示例 1:
```
输入: [1,2,3]
输出: [1,2,4]
```
解释: 输入数组表示数字 123。
示例 2:
```
输入: [4,3,2,1]
输出: [4,3,2,2]
```
解释: 输入数组表示数字 4321。


```java
fun plusOne(digits: IntArray): IntArray {
    for (index in digits.size - 1 downTo  0) {
        digits[index]++
        digits[index] = digits[index] % 10
        if (digits[index] != 0) return digits
    }

    val newDigits = IntArray(digits.size + 1)
    newDigits[0] = 1
    return newDigits

}
```
## 70 怕楼梯

假设你正在爬楼梯。需要 n 阶你才能到达楼顶。

每次你可以爬 1 或 2 个台阶。你有多少种不同的方法可以爬到楼顶呢？

注意：给定 n 是一个正整数。

示例 1：

```
输入： 2
输出： 2
解释： 有两种方法可以爬到楼顶。
1.  1 阶 + 1 阶
2.  2 阶
```

示例 2：

```
输入： 3
输出： 3
解释： 有三种方法可以爬到楼顶。
1.  1 阶 + 1 阶 + 1 阶
2.  1 阶 + 2 阶
3.  2 阶 + 1 阶
```

使用递归的方式
```java
fun climbStairs(n: Int): Int {
    if (n == 1) return 1
    if (n == 2) return 2
    return climbStairs(n - 1) +  climbStairs(n - 2)
}
```

feibonaqie 数列
```java
    fun climbStairs(n: Int): Int {
    if (n == 1) return 1
    var i = n
    var head = 1;
    var tail = 2

    while (i >= 3) {
        val temp = head + tail
        head = tail
        tail = temp
        i--
    }
    return tail
    }
```


## 88. 合并两个有序数组

给你两个有序整数数组 nums1 和 nums2，请你将 nums2 合并到 nums1 中，使 nums1 成为一个有序数组。

 

说明:

初始化 nums1 和 nums2 的元素数量分别为 m 和 n 。
你可以假设 nums1 有足够的空间（空间大小大于或等于 m + n）来保存 nums2 中的元素。
 
```
示例:

输入:
nums1 = [1,2,3,0,0,0], m = 3
nums2 = [2,5,6],       n = 3

输出: [1,2,2,3,5,6]
```

使用双指针法，从后开始遍历。当其中一个到达-1时，将剩余的平移过去
```java
fun merge(nums1: IntArray, m: Int, nums2: IntArray, n: Int): Unit {
    var p1 = m-1;
    var p2 = n-1;
    var p = m+n -1;

    while (p1>=0 && p2>= 0){
        nums1[p--] = if (nums1[p1] <nums2[p2]) nums2[p2--] else nums1[p1--]
    }

    while (p2>=0){
        nums1[p--] = nums2[p2--]
    }

    while (p1>=0){
        nums1[p--] = nums1[p1--]
    }
}
```

## 移动零
给定一个数组 nums，编写一个函数将所有 0 移动到数组的末尾，同时保持非零元素的相对顺序。

示例:
```
输入: [0,1,0,3,12]
输出: [1,3,12,0,0]
```
说明:
- 必须在原数组上操作，不能拷贝额外的数组。
- 尽量减少操作次数。


双指针，一个遍历指针一个目标地址指针。
```java
fun moveZeroes(nums: IntArray): Unit {
    var i =0
    var j =0;
    while (i != nums.size){
        if (nums[i]!=0){
            nums[j++] = nums[i]
        }
        i++
    }

    while (j!=i){
        nums[j++] =0
    }

}
```
