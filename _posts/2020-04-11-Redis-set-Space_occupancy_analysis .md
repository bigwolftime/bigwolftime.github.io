---
layout: post
title: Redis set 空间占用分析
date: 2020-04-11
Author: bwt
categories: Redis
tags: [Redis]
comments: true
---

> Redis version 2.8.20 OS: 64bit

#### 一
Redis 中存储字符串的结构为 SDS(简单动态字符串), 其定义包括:

```c
struct sdshdr {
    unsigned int len; // 字符串占用总长度
    unsigned int free; // 剩余可用长度
    char buf[];		// 存储实际的数据, 最后一位追加'\0'为结束标记
};
```

所以 SDS 占用空间为: `4(len) + 4(free) + n + 1('\0') = 9 + n(单位:byte, n 为字符串长度)`

SDS 是对字符串的一个封装, 在其之上会封装一层 `RedisObject`, 其定义包括:

```c
typedef struct redisObject {
    unsigned type:4;    // (:4代表占用 4bit)
    unsigned encoding:4;
    unsigned lru:REDIS_LRU_BITS; // REDIS_LRU_BITS = 24
    int refcount;	// 引用计数, 涉及到空间释放时会用到, 若不为 0 则不能被释放
    void *ptr;		// 此处指针指向实际的的 SDS 地址
} robj;
```

RedisObject 占用空间: `(4 + 4 + 24) / 8 + 4 + 8 = 16byte`
所以总结出一个完整的 RedisObject 对象总空间占用为: `9 + n + 16 = n + 25`

其中: Redis 的 `tryObjectEncoding` 函数会判断指针 ptr 所指向的 SDS 能否转换为整数, 若可以 ptr 直接将其保存为整数, 并释放掉刚才的 SDS. 
即: 当 value 可转化为整数时, 总占用为 16byte, 否则为 (n + 25)byte

#### 二
当执行 `set aaa bbb` 时, 空间占用分析:
首先 "set" "aaa" "bbb" 会各生成 1 个 RedisObject, 各自占用 (25 + 3)byte.
其中 "set" 用来查找命令 `lookupCommand`, 完成之后会释放掉`RedisObject` 结构(仅保留 SDS), "aaa" 这个对象也会用 SDS 存储, 也会释放掉 `RedisObject` 结构.
所以目前总占用为: `12(SDS(set)) + 12(SDS(aaa)) + 28(RedisObject(bbb)) = 52byte`

最后这组键值对最终会存储到`dictEntry` 结构中, 其定义为:

```c
typedef struct dictEntry {
    void *key;	// 即上面的 "aaa" 的 SDS 指针
    union {
        void *val;	// 指向 RedisObject 的指针
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next; // 下一个节点的指针
} dictEntry;
```

`dictEntry` 的占用为: `8(key) + 8(val) + 8(next) = 24byte`

总占用为: `24(dictEntry) + 12(SDS(aaa)) + 28(RedisObject(bbb)) = 64byte`
若执行的 value 为整数, 总占用为: `24(dictEntry) + 12(SDS(aaa)) + 16(RedisObject(16)) = 52byte`

#### 参考

[Redis 一组 kv 实际内存占用计算](https://kernelmaker.github.io/Redis-StringMem)