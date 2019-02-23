---
id: "2019-01-02-17-15"
date: "2019/01/02 17:15"
title: "Q807 保持城市天际线(Max Increase to Keep City Skyline)"
tags: ["java", "leetcode","array",'leetcode']
categories: 
- "算法"
- "leetcode刷题"
---

### 解析思路

&emsp;&emsp;leetcode 中等难度中比较简单的一个，题目描述[看这里](https://leetcode-cn.com/problems/max-increase-to-keep-city-skyline/),最开始看题目是比较懵逼的，第一遍没有看懂，多看几次才明白，其实就是让每个点增大到不超过本行且本列的最大值，输出增加的值的和。so 解题方法就是算出每行每列的最大值，然后每个点与当前行当前列最大值中较小的一个差值的和就是结果。

### 代码(Java实现)

```java
class Solution {
    public int maxIncreaseKeepingSkyline(int[][] grid) {
        int rowLength = grid.length;
        int columnLength = grid[0].length;
        //每行对应的最大值
        int[] rowMax = new int[rowLength];
        //每列对应的最大值
        Arrays.fill(rowMax, -1);
        int[] columnMax = new int[columnLength];
        Arrays.fill(columnMax, -1);
        int sum = 0;
        for (int i = 0; i < rowLength; i++) {
            for (int j = 0; j < columnLength; j++) {
                if (rowMax[i] == -1) {
                    //计算行最大值
                    for (int temp = 0; temp < columnLength; temp++) {
                        if (rowMax[i] < grid[i][temp]) {
                            rowMax[i] = grid[i][temp];
                        }
                    }
                }
                if (columnMax[j] == -1) {
                    //计算列最大值
                    for (int temp = 0; temp < rowLength; temp++) {
                        if (columnMax[j] < grid[temp][j]) {
                            columnMax[j] = grid[temp][j];
                        }
                    }
                }
                sum += Math.min(rowMax[i], columnMax[j]) - grid[i][j];
            }
        }
        return sum;
    }
}
```
