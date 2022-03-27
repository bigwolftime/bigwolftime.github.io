---
layout: post
title: Android SSL 请求抓包
date: 2020-03-06
Author: bwt
categories: 笔记
tags: [Packet Capture, SSL, Android, root]
comments: true
---

**提示: Android 设备需获取 root 权限, 不同的厂商或者型号有不同的 root 操作方法, 非介绍重点**

**前提: 设备开启"开发人员选项", 并开启 USB 调试(重要)**

<!--break-->

Android 7.0 版本之后默认不信任用户证书, 一些 SSL 请求由于证书不被信任, 导致数据无法解密, 解决的办法:
1. 换用低版本的 Android 设备;
2. 设备 root, 然后将用户证书 push 到系统证书目录下, 即可达到目的.


```shell
# adbd 以 root 权限执行
$ adb root

# 将 /system 目录置于可读写模式
$ adb remount
# 在这一步可能会出现 'Read-only file system' 错误, 如果有的话执行:
$ adb disable-verity (关闭在调试环境下的dm-verity检查)
# 有如下输出, 按照提示重启, 重新执行前两条命令即可
Successfully disabled verity
Now reboot your device for settings to take effect

# 将用户证书目录的文件 复制到 系统信任证书目录
adb shell cp -f /data/misc/user/0/cacerts-added/* /system/etc/security/cacerts/
```

##### 远程调试方法

> 远程调试即在 PC, Android 手机在同一 Wifi 下, 无需数据线实现 adb 连接.

远程调试连接可分为两种:
1. 借助 USB 数据线, 更改 adb 监听端口之后即可断开 USB;
2. 全程无需 USB 连接(**需要设备 root**)
不过这两种办法的原理是一样的, 都是更改设备的 adb 监听端口. 
若由设备自行更改, 则需要 root 权限才行. 看来 方法1 是比较优雅的.

###### 1. 借助 USB 线

```shell
# 有线连接成功之后, 执行:
adb tcpip 5555

# 到此就完成了, 回到 PC 执行:

adb connect ip(设备的 ip 地址, 可以在 设置 - 关于手机 - 状态 找到相关信息)
```

###### 2. 无需 USB 线

手机安装一个 shell 终端软件, 例如: [Ansole](https://www.coolapk.com/apk/com.romide.terminal), Terminal.

```shell
# 手机打开终端软件, 按照以下步骤:
su # 获取 root 权限
setprop service.adb.tcp.port 5555    //设置 adbd 监听端口

# 重启 adbd 服务
stop adbd
start adbd

# 查看本机 IP 地址, 需 PC / 手机在同一 Wifi.
ip addr

# 回到 PC. 输入:
adb connect ip(上一步得到的手机 ip)

# 没有问题的话则连接成功, 查询连接的设备信息
adb devcices

```

##### 参考

1. [ADB 用法教程](https://github.com/mzlogin/awesome-adb)
1. [Android 8.0 Fiddler 抓包](https://bbs.pediy.com/thread-256867.htm)
2. [adb push 失败提示 ‘Read-only file system](https://www.jianshu.com/p/eca9a8ad4996)