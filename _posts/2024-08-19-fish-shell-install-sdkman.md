---
layout: post
title: Fish shell 安装 sdkman
date: 2024-08-19
Author: bwt
categories: shell
tags: [sdkman]
comments: true
toc: true
---

SDKMAN 是一个跨平台的工具，可以帮助你管理多个版本的 Java 以及其他开发工具（如 Maven、Gradle 等）。

<!--break-->

在 fish shell 中安装和使用 SDKMAN 需要一些额外的步骤，因为 SDKMAN 的安装脚本默认是为 bash 和其他一些 shell 设计的。以下是如何在 fish
shell 中安装和配置 SDKMAN 的详细步骤：

### 1. 使用 brew 安装 sdkman

```shell
brew tap SDKMAN/tap
brew install SDKMAN-cli
```

### 2. 安装 fisher

`fisher` 是一个用于 `fish shell` 的插件管理器.

```shell
curl -sL https://raw.githubusercontent.com/jorgebucaran/fisher/main/functions/fisher.fish | source && fisher install jorgebucaran/fisher
```

### 3. 使用 fisher 安装 sdkman

```shell
# 可以通过 https://github.com/reitzig/sdkman-for-fish/releases/ 找到最新的版本号
fisher install reitzig/SDKMAN-for-fish@v2.1.0
```

### 4. 配置

找到 sdkman 的安装位置:

```shell
echo $(brew --prefix SDKMAN-cli)/libexec
# 在我的电脑上, 输出结果是: /usr/local/opt/SDKMAN-cli/libexec

touch ~/.config/fish/conf.d/config_sdk.fish

set -g __sdkman_custom_dir /usr/local/opt/SDKMAN-cli/libexec
```

至此已经准备完成.

### 5. 参考

- [Setting up SDKMAN! with Fish and Homebrew](https://www.thinkbinary.co.uk/2024/01/07/setting-up-sdkman-with-fish-and-homebrew)