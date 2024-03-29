---
id: "20210326"
date: "2021/03/26 17:15"
title: "Q1663 具有给定数值的最小字符串(Smallest String With A Given Numeric Value)"
tags: ["java", "leetcode"]
categories:
  - "算法"
  - "leetcode刷题"
---


### 解析思路

&emsp;&emsp;leetcode 中等难度中比较简单的一个，题目描述[点击这里](https://leetcode-cn.com/problems/smallest-string-with-a-given-numeric-value/)。读完描述可将本题精简为如下内容：

---

给两个整数 n 和 k,返回序列长度为 n 且数字和等于 k 的一个数字序列(每个数字的范围为 1-26,对应 26 个字母)，要求小的数字尽量放前面.

---

&emsp;&emsp;看到尽量小的数字放在前面且数字和是固定的，我们就应该想到可以用贪心算法来解决这个问题，思路如下：
设定 i=1,s=1

1. 第 i 个数字放入 s,假设后面数字全部为 26,判断剩下的数字还能否满足要求
2. 如数字和&lt;k 说明无法满足，s=s+1 重复第一步
3. 如数字和>=k 说明能否满足，i=i+1,s=s+1,重复第一步
4. 直到所有的数字都填入

<!-- more -->

java 代码见：[点击这里](https://github.com/FleyX/demo-project/blob/master/5.leetcode/src/com/fanxb/interview/Q1663.java),translateNum1 方法

进一步思考会发现上面的解法存在存在很多的循环，效率不高，能否优化？

当然可以，我们并不需要每次+1 后再判断能否满足需求，一次计算即可计算出当前位置最小能填入多少，流程如下：设定 i=1,sum=0

1. 假设 i 以后的位置全填入 26,计算出还缺多少才能补足到 k. temp=(26\*(n-i))-(k-sum)
2. 如果 temp>=0 说明后面全填 26 肯定能满足要求，因此当前位置填入最小值 1,i=i+1,sum=sum+1,重复 1
3. 如果 temp&lt;0 说明即使后面全为 0 也不能满足要求，需要在 i 填入-temp 的值才行，i=i+1,sum=sum+（-temp),重复 1

java 代码见：[点击这里](https://github.com/FleyX/demo-project/blob/master/5.leetcode/src/com/fanxb/interview/Q1663.java),translateNum 方法

本文解法是将尽量小的数字填到前面，另外一种思路正好相反，将尽量大的数字填到后面，可自行尝试。

---

另外本体可换一种描述，要求数字序列拼成的数字最小，比如['12','32']拼成 1232,也是一样的解法。

---


<span id="blogIdSpan" style="display:none">20210326</span>

