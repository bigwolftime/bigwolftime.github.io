---
layout: post
title: TCP 协议
date: 2020-11-07
Author: bwt
categories: 计算机网络
tags: [计算机网络, TCP]
comments: true
toc: true
---

#### 一. TCP 报文格式

![TCP 报文格式](https://zonheng.net/tcp_form.png)

tcp flags 解释:

* URG: 报文中含有紧急数据, 需提高优先级传送;
* PSH: 是否带有 push 标记(1代表是),
  https://honeypps.com/network/tcp-introduction/