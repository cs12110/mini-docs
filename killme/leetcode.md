# Leetcode

泪流满面.

---

## 两数之和

### 题目

```
给定一个整数数组 nums 和一个目标值 target，请你在该数组中找出和为目标值的那 两个 整数，并返回他们的数组下标。

你可以假设每种输入只会对应一个答案。但是，你不能重复利用这个数组中同样的元素。

示例:

给定 nums = [2, 7, 11, 15], target = 9

因为 nums[0] + nums[1] = 2 + 7 = 9
所以返回 [0, 1]
```

### 答案

自己写的如下,惨不忍睹

```java
public int[] twoSum(int[] nums, int target) {
    int[] arr = new int[2];
    for (int i = 0; i < nums.length; i++) {
        for (int j = i + 1; j < nums.length; j++) {
            if (i != j && nums[i] + nums[j] == target) {
                arr[0] = i;
                arr[1] = j;
                break;
            }
        }
    }
    return arr;
}
```

是时候,看一下江湖是多大的了.

```java
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> map = new HashMap<>(nums.length);
    for (int i = 0, len = nums.length; i < len; i++) {
        int tmp = target - nums[i];
        if (map.containsKey(tmp)) {
            return new int[]{map.get(tmp), i};
        }
        map.put(nums[i], i);
    }
    return null;
}
```

## 回文数

### 题目

```
判断一个整数是否是回文数。回文数是指正序（从左向右）和倒序（从右向左）读都是一样的整数。

示例 1:

输入: 121
输出: true
示例 2:

输入: -121
输出: false
解释: 从左向右读, 为 -121 。 从右向左读, 为 121- 。因此它不是一个回文数。
示例 3:

输入: 10
输出: false
解释: 从右向左读, 为 01 。因此它不是一个回文数
```

### 答案

终于自己做出了一个提交得到不错的东西,泪目.

```java
public boolean isPalindrome(int x) {
    String str = String.valueOf(x);
    char[] chars = str.toCharArray();
    for (int index = 0, limit = chars.length / 2; index < limit; index++) {
        if (chars[index] != chars[chars.length - index - 1]) {
            return false;
        }
    }
    return true;
}
```

## 括号配对

```
给定一个只包括 '('，')'，'{'，'}'，'['，']' 的字符串，判断字符串是否有效。

有效字符串需满足：

左括号必须用相同类型的右括号闭合。
左括号必须以正确的顺序闭合。
注意空字符串可被认为是有效字符串。

示例 1:

输入: "()"
输出: true
示例 2:

输入: "()[]{}"
输出: true
示例 3:

输入: "(]"
输出: false
示例 4:

输入: "([)]"
输出: false
示例 5:

输入: "{[]}"
输出: true
```

### 答案

```java

```
