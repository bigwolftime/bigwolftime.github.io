---
layout: post
title: Android app SSL 请求抓包
date: 2020-03-06
Author: bwt
categories: 笔记
tags: [抓包, SSL, Android, root]
comments: true
toc: true
---

本文介绍三种抓包方案.

**提示: Android 设备可能需要获取 root 权限, 不同的厂商或者型号有不同的 root 操作方法, 非介绍重点**

<!--break-->

### 一. system 分区读写

我们的目标是将用户证书目录下的文件复制到系统证书目录, 但默认情况下 Android 系统的 system 分区为只读, 需要我们挂载为读写. 

根据使用的设备进行分类, 可分为: 模拟器或者真机(root).

* 模拟器抓包方案最简单, 无需真机. 但多数模拟器的 Android 版本都比较老(基本为 7.0 及以下), 一些 app 可能会不兼容导致闪退, 再或者某些 app 
会主动检测运行环境, 发现模拟器特征也会拒绝运行.

* 使用真机, 具体情况可分为以下三种:

#### 1. Android 7.0 以下

这种场景最简单, 无需特殊配置.

#### 2. Android 7.0+

Android 7.0+ 系统默认不再信任用户证书, 因此一些 https 请求由于证书不被信任, 导致链接建立失败, 需要我们获取到设备 root 权限, 然后挂载 
system 分区为读写, 操作命令如下:

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
```
若以上命令无报错, 说明 system 分区挂载读写成功.

#### 3. Android 10+

高版本的 Android 即使拥有 root 权限, 也不能通过上面的命令达到挂载 system 分区为可读写的目的, 具体表现可能是这样的:

```shell
$ adb root
adbd cannot run as root in production builds

$ adb remount
/system/bin/sh: remount: inaccessible or not found

$ adb disable-verity
/system/bin/disable-verity only works for userdebug builds
verity cannot be disabled/enabled - USER build
```

如果使用了 Magisk root 方案, 可以通过安装 Magisk 模块: [Magical Overlayfs](https://github.com/HuskyDG/magic_overlayfs/)达到目的.


如果这些准备完成, 可以看下一步.

### 二. 信任用户证书

可以使用命令或者 MT管理器 将用户证书复制到系统证书目录下, 参考 shell:

`adb shell cp -f /data/misc/user/0/cacerts-added/* /system/etc/security/cacerts/`

真机 Android 7.0+ 若想上面的命令不报错, 需要保证 system 分区具有读写权限, 就是步骤一所做的工作.

### 题外话

记录下远程调试的方法(保证 PC 和 Android 设备在同一局域网):

1. 借助 USB 数据线, 执行以下 shell:

```shell
# 有线连接成功之后, 执行:
adb tcpip 5555

# 断开 USB 连接后 PC 执行:
adb connect youtIPAddress
```

2. root + 无需 USB

Android 设备安装 shell 终端软件, 例如: [termux/termux-app: Termux - a terminal emulator application for Android OS 
extendible by variety of packages.](https://github.com/termux/termux-app), 然后在终端执行:

```shell
su # 获取 root 权限
setprop service.adb.tcp.port 5555    //设置 adbd 监听端口

# 重启 adbd 服务
stop adbd
start adbd
```

完成后 PC 端执行 `adb connect ip` 即可.

3. adb pair

> 不同的系统可能有不同的操作路径, 此处以`XiaoMi 13`为例:
> Android 设备开启"开发者选项", 进入开发者选项后找到"无线调试", 打开调试开关, 点击"使用配对码配对设备".
> PC 端执行: `adb pair IP:port`, 然后输入设备上的配对码即可.

![adb pair code](http://zonheng.net/tech/adb_pair_code.jpg-thumbnail)


### 参考

* [ADB 用法教程](https://github.com/mzlogin/awesome-adb)
* [Android 8.0 Fiddler 抓包](https://bbs.pediy.com/thread-256867.htm)
* [adb push 失败提示 ‘Read-only file system](https://www.jianshu.com/p/eca9a8ad4996)
* [SDK Platform Tools 版本说明](https://developer.android.com/studio/releases/platform-tools?hl=zh-cn)
* [Magical Overlayfs](https://github.com/HuskyDG/magic_overlayfs/)