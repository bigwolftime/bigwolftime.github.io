---
layout: post
title: Android https 请求抓包(透明代理)
date: 2023-08-10
Author: bwt
categories: 笔记
tags: [抓包, SSL, Android, root]
comments: true
toc: true
---

有些 app 会对 Wi-Fi 代理的进行检测, 如果命中, 则任何请求都会执行失败, 抓包软件也看不到任何请求. 既然如此, 有什么办法可能绕过 app 的检测呢? 
此处引入就需要引入`透明代理`.

顾名思义, `透明代理`即让 app 感觉不到自己正在被代理(此种方式仍然需要设备拥有 root 权限).

有兴趣可以阅读之前的基础文章: [2020-03-06-Android-SSL-packet.md](./2020-03-06-Android-SSL-packet.md)

<!--break-->

> 阅读了 [frida-skeleton](https://github.com/Margular/frida-skeleton) 项目的源码, 受到启发, 即通过 iptables 配置将端口的流量转发
> 到代理节点, 上网搜索发现已经有许多技术大佬做了实现, 此处做下记录.

### 一. 配置 Magisk 插件

需要两个 Xposed/LSPosed 插件: [JustMePlush 看雪论坛大神杰作](https://bbs.kanxue.com/thread-254114.htm) 和 
[TrustMeAlready](https://github.com/ViRb3/TrustMeAlready), 下载完成后将要抓包的 app 打上勾.

> SSLUnpinning 也使用过, 但是抓包没有效果, 不知是不是使用姿势不对. :dog:

### 二. 配置 Android 设备流量转发

关键操作是操作 Android 设备, 将所有经过 80, 443 端口的流量都走到代理节点:

```shell
# 进入 Android 设备 shell
adb shell
# 获取超级权限
su
# 配置端口流量转发
iptables -t nat -A OUTPUT -p tcp --dport 80 -j DNAT --to  代理ip:80
iptables -t nat -A OUTPUT -p tcp --dport 443 -j DNAT --to  代理ip:443
```

执行完成后, 可以通过命令查看下是否符合预期:

```shell
iptables -t nat -L | grep -E '443|80'

# 预期的输出(192.168.25.157 是局域网代理 IP 地址):
target     prot opt source               destination
----------------------------------------------------------------
DNAT       tcp  --  anywhere             anywhere             tcp dpt:https to:192.168.25.157:443
DNAT       tcp  --  anywhere             anywhere             tcp dpt:http to:192.168.25.157:80
```

### 三. 代理节点设置

此处需要使用 BurpSuite 软件, 配置监听 443 和 80 端口的所有流量.

首先创建 443 端口的监听策略:

![burp_443](https://zonheng.net/tech/burp_443.png-thumbnail)

![burp_443_bind](https://zonheng.net/tech/burp_443_bind.png-thumbnail)

然后创建 80 端口的监听策略:

![burp_80](https://zonheng.net/tech/burp_80.png-thumbnail)

![burp_80_bind](https://zonheng.net/tech/burp_80_bind.png-thumbnail)

最终创建好的监听策略如图:

![burp_bind](https://zonheng.net/tech/burp_bind.png-thumbnail)

### 四. 实践

以某影视 app 为例, 可以看到请求详情:

![实践](https://zonheng.net/tech/burp_catch.png-thumbnail)

抓包完成后需要将 iptables 配置还原, 否则断开 Wi-Fi 连接或者关闭代理节点后, Android 设备无法访问网络, 删除的命令:

```shell
iptables -t nat -D OUTPUT -p tcp --dport 80 -j DNAT --to  代理ip:80
iptables -t nat -D OUTPUT -p tcp --dport 443 -j DNAT --to  代理ip:443

# 此时若执行下面的 shell 应该返回空的结果:
iptables -t nat -L | grep -E '443|80'
```

### 五. 参考

* [frida-skeleton](https://github.com/Margular/frida-skeleton)
* [浅析APP代理检测对抗](https://xz.aliyun.com/t/11398)
* [(原创)自识别类名 自动化Hook JustTrustMe 升级版](https://bbs.kanxue.com/thread-254114.htm)
* [TrustMeAlready](https://github.com/ViRb3/TrustMeAlready)