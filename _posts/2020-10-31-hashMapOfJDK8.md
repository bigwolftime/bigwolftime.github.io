---
layout: post
title: HashMap of JDK8
date: 2020-04-20
Author: bwt
categories: Java
tags: [Java, HashMap]
comments: true
toc: true
---

### 一. 概要

JDK8 版本的 HashMap 相对于 JDK7 有了一些优化, 比较明显的就是引入了红黑树的数据结构, 提升查询效率.

![HashMap的类继承关系图](https://zonheng.net/map.jpg)

1. HashMap:

日常使用频次较高的结构. 非线程安全, 如果有线程安全的需求, 可以考虑使用 `Collections.synchronizedMap()` 或者 ConcurrentHashMap.
允许 key 为 null(最多只能有一个), 允许 value 为 null.
键值对的插入顺序与遍历顺序不固定.

2. HashTable:
遗留类，很多映射的常用功能与 HashMap 类似，不同的是它承自 Dictionary 类，并且是线程安全的. 
任一时间只有一个线程能写 Hashtable，并发性不如 ConcurrentHashMap，因为 ConcurrentHashMap 引入了分段锁.

3. LinkedHashMap:
属于 HashMap 的子类, 底层使用链表实现, 可保存键值对的插入顺序.

4. TreeMap:
实现 SortedMap 接口, 默认根据"键"进行排序(也可自定义比较器).

### 二. HashMap 的实现

![JDK8 HashMap 的数据结构](https://zonheng.net/aaaa.png)

图中有一个数组 table, 当键值对插入时, 会根据 key 进行 hashCode() 计算, 然后一系列操作后确定数组的索引位置(下面介绍), 然后键值对会封装成 
Node 对象存储到 table 数组中. Node 的代码如下:

#### 1. Node

```java
// Node 是 HashMap 内部类, 实现 Entry 接口
static class Node<K, V> implements Entry<K, V> {
        final int hash;
        final K key;
        V value;

        /**
        *  下一个元素的地址. 用于解决 hash 冲突
        *  hash 冲突的解决办法一般有: 链地址法和开放地址法(我所见过的一些组件, hash 冲突使用的基本都是链地址, 开放地址法暂未遇到过)
        *
        *  hash 冲突一般与这些因素有关:
        *   1. hash 函数: 一个好的 hash 函数, 基本可以保证数据均匀分布, 冲突最少, 以达到最好的性能;
        *   2. table 数组容量: 若容量较小, 即使使用优秀的 Hash 函数, 也会有大概率的冲突.
        */
        HashMap.Node<K, V> next;

        Node(int hash, K key, V value, HashMap.Node<K, V> next) {}
        public final K getKey() {}
        public final V getValue() {}
        public final String toString() {}
        public final int hashCode() { return Objects.hashCode(this.key) ^ Objects.hashCode(this.value); }
        public final V setValue(V newValue) {}
        public final boolean equals(Object o) {
            if (o == this) {
                return true;
            } else {
                if (o instanceof Entry) {
                    Entry<?, ?> e = (Entry)o;
                    if (Objects.equals(this.key, e.getKey()) && Objects.equals(this.value, e.getValue())) {
                        return true;
                    }
                }

                return false;
            }
        }
    }
```

#### 2. 初始化参数

```java
// 默认的 table 数组容量
static final int DEFAULT_INITIAL_CAPACITY = 16;
// 默认的负载因子值
static final float DEFAULT_LOAD_FACTOR = 0.75F;

/**
* 当链表长度较长, 那么查询的时间复杂度会变为 O(N), 性能下降. 
* 所以此处做出了优化: 当长度达到 8 则转换为红黑树, 时间复杂度降低到 O(logN)
*/
static final int TREEIFY_THRESHOLD = 8;

// 实际的 Node 个数(注意和 table.length, threshold 区别)
transient int size;
// 记录 HashMap 内部结构发生变化的次数, 用于迭代的快速失败.
transient int modCount;

// 当前可容纳键值对的最大值
int threshold;

/**
*  负载因子. 计算方式为: loadFactor = threshold(注意: 非实际的 Node 个数) / table.length
*  当负载因子过大, 即表示填充程度越高, hash 冲突的概率也会增大, 此时需要扩充数组, 以维持其性能.
*/
final float loadFactor;
```

此外, 数组的长度也有要求, 需要保证为 `table.length = 2 的 n 次幂`, 采用这种设计主要是优化: 取模和扩容的过程.

#### 3. 确定桶索引位置

对 map 中的 key 进行曾删改查, 第一步就是要根据 key 确定数组的哪个索引位置.

```java
static final int hash(Object key) {
    int h;
    return key == null ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// jdk1.7的源码，jdk1.8没有这个方法，但是实现原理一样的
static int indexFor(int h, int length) {
	/**
     * 由于取模计算消耗较大, 这里可以用 h & (length-1) 代替 (h % length). 
     * 此处就是数组长度是 2 的幂次的好处, 恰好满足这个规律, 非常巧妙.
     */
	return h & (length-1);
}
```

上面的 `hash()` 方法: `(h = key.hashCode()) ^ (h >>> 16)`. 首先计算 key 的 hashCode(int_32), 然后和 hashCode 的高 16bit 异或计算, 
当数组的长度较小, 也可保证高低位都参与到计算中, 同时不会有大的开销. 如下图:

![hash 的计算与索引的确定过程举例](https://pic2.zhimg.com/8e8203c1b51be6446cda4026eaaccf19_r.jpg)

#### 4. put 方法


### 参考

* [Java 8系列之重新认识HashMap](https://zhuanlan.zhihu.com/p/21673805) 2016
* [为什么HashTable的桶会取一个素数](https://blog.csdn.net/liuqiyao_01/article/details/14475159) 2013
* [教你初步了解红黑树](https://blog.csdn.net/v_july_v/article/details/6105630) 2010
* [JDK 笔记](https://github.com/seaswalker/JDK/blob/master/note/HashMap/hashmap.md)