---
layout: post
title: 排序算法
date: 2020-02-03
Author: bwt
categories: 数据结构
tags: [Algorithm, Data Structure, Sort]
comments: true
---

#### 一. 衡量算法的好坏

衡量排序算法的好坏一般从这几个角度分析:

##### 1. 最好, 最坏, 平均时间复杂度

衡量算法的优劣首先离不开时间复杂度的分析, 另外对于不同规律的原始数据, 算法的性能表现也有不同. 例如乱序和完全有序的两组数据, 对执行时间也是有影响的.

##### 2. 空间复杂度



##### 3. 稳定性

稳定性也是取舍算法的一项重要指标. 所谓稳定性是说: 序列中若存在等值元素, 排序之后等值元素的**相对位置**不变.

例如: `4, 2(1), 3, 2(2), 6`. (括号中用来表示不同的2). 若保证稳定性, 则排序后应为: `2(1), 2(2), 3, 4, 6`. 即两个2的相对位置没有变化.

若算法可满足此特性, 则称之为**稳定性的排序算法**, 反之则称为**不稳定的排序算法**.

###### 3.1 上面的例子可以理解稳定性的概念, 那么实际的应用中稳定性有什么意义呢?

在实际的开发中, 单纯地比较数字大小的情况较少, 多数情况是提供一组对象, 根据对象的某些字段进行排序.
例如: User(Integer id, Integer age) 对象. 假如定义这样的排序规则: 根据 age 升序, 当 age 相同时按 id 降序.
原始数据如下:

|  id  | age  |
| :---: | :---: |
|  1   |  24  |
|  2   |  21  |
|  3   |  18  |
|  4   |  15  |
|  5   |  21  |

首先根据 id 降序:

|  id  | age  |
| :---: | :---: |
|  5   |  21  |
|  4   |  15  |
|  3   |  18  |
|  2   |  21  |
|  1   |  24  |

然后使用**稳定**排序, 根据 age 升序排列:

|  id  | age  |
| :---: | :---: |
|  4   |  15  |
|  3   |  18  |
|  5   |  21  |
|  2   |  21  |
|  1   |  24  |

要验证上面的思路, 可以手动创建一张表, 执行: `select * from table order by age asc, id desc`, 可以得到相同的结果

```text
+----+-----+
| id | age |
+----+-----+
| 4  | 15  |
| 3  | 18  |
| 5  | 21  |
| 2  | 21  |
| 1  | 24  |
+----+-----+
5 rows in set
Time: 0.165s
```

##### 4. 比较 / 交换的次数

排序算法设计到的操作主要是: 比较和交换, 需要把这部分也考虑进来.


#### 二. 排序


##### Arrays.sort() 解读


#### 参考
1. https://time.geekbang.org/column/article/41802