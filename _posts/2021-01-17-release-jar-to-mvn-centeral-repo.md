---
layout: post
title: 发布 Jar 包到 Maven 中央仓库
date: 2021-01-17
Author: bwt
categories: 其他
tags: [Maven Central Repo]
comments: true
toc: true
---

本文将介绍将 Jar 包构建并发布到 Maven Central Repo 的过程.

<!--break-->

#### 一. 创建 sonatype 账号

*已有可忽略此步骤*

地址: https://issues.sonatype.org/secure/Signup!default.jspa

#### 二. 创建 Issue

在 [https://issue.sonatype.org](https://issue.sonatype.org) 网页中, 点击上方菜单的 `Create`, 填写要发布项目的主要信息:

* Project: 选择 `Community Support - Open Source Project Repository Hosting (OSSRH)`;
* Issue Type: 可以选择 `New Project`;
* Summary: 填写项目的介绍信息;
* group Id: 为组织或者团队的标识, 可以通过自定义域名方式命名, 例如: `com.abc`, `cn.abc` 等, 或者通过 github 方式命名: 
  `io.github.userName` 或者 `com.github.userName`, 需要注意此处不要随便取名, 因为会涉及到后面的 groupId 所属认证.
* Project Url: 项目介绍网站, 此处可以填写上面自定义的域名或者 github 项目主页.
* SCM Url: 项目的源代码地址(Location of source control system, e.g. https://github.com/sonatype/nexus-oss.git)

填写完成后提交, 会生成一个格式为: `OSSRH-xxxxxx` 的 issueId.

#### 三. groupId 认证

当工作人员收到 Issue 后, 会在你提交的 issue 下新增一条评论, 内容大致为: 要求进行 groupId 认证, 以证明自定义域名或者 github 账号属于你. 
不同风格的 groupId 有不同的认证方式.

1. 如果 groupId 为自定义域名风格, 认证方式一般为添加 DNS 的 TXT 记录, 指向此 issue 地址(`原文: Add a TXT record to your DNS 
   referencing this JIRA ticket: OSSRH-xxxxxx`), 或者将域名重定向到 github 的项目地址(`原文: Setup a redirect to your 
   Github page(if it does not already not)`), 二者选其一即可;
2. 如果使用 github 命名方式, 则需要在对应 userName 的 github 账号下创建一个名为: `OSSRH-xxxxxx` 的公共仓库(`Please create a 
   public repo called OSSRH-xxxxx to verify github account ownership`)

完成之后, 同样以添加一条评论的方式通知工作人员.

#### 四. 完善项目信息

以上完成之后, 若没有问题, 会得到以下回复:

> groupId has been prepared, now user(s) can:
>
> Deploy snapshot artifacts into repository https://oss.sonatype.org/content/repositories/snapshots
> 
> Deploy release artifacts into the staging repository https://oss.sonatype.org/service/local/staging/deploy/maven2
> 
> Release staged artifacts into repository 'Releases'
> 
> please comment on this ticket when you promoted your first release, thanks

下一步可以将 maven 的仓库地址配置到项目中:

```xml
<distributionManagement>
        <repository>
            <id>oss-repo</id>
            <name>Central Repo OSSRH</name>
            <url>https://oss.sonatype.org/service/local/staging/deploy/maven2/</url>
        </repository>
        <snapshotRepository>
            <id>oss-repo</id>
            <url>https://oss.sonatype.org/content/repositories/snapshots</url>
        </snapshotRepository>
    </distributionManagement>
```

此外还需配置 maven 的 setting.xml 文件, 配置仓库的用户名及密码, 就是第一步的用户名 & 密码

```xml
<servers>
    <server>
        <id>oss-repo</id>
        <username>yourUserName</username>
        <password>yourPassword</password>
    </server>
</servers>
```
**注意: setting.xml 文件中 server - id 与 pom.xml 文件中 distributionManagement - repository - id 要保持一致**

除此之外还有其他要求, 需要在 `pom.xml` 文件中以标签的方式声明, 具体可见[参考-2].


#### 五. GPG

*此步骤具体可见[参考3].*

所有被部署的文件都需要开发者使用 GPG 进行签名以验证安全性.

1. 需要安装 gpg 工具, 安装过程略.
2. 生成密钥对 `gpg --gen-key`, 按提示生成即可;
3. 使用 `gpg --list-keys` 命令可查看已生成的密钥对;

   ```shell
   $ gpg --list-keys
   
   gpg: checking the trustdb
   gpg: marginals needed: 3  completes needed: 1  trust model: pgp
   gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
   gpg: next trustdb check due at 2023-01-17
   ---------------------------------
   pub   rsa2048 2021-01-17 [SC] [expires: 2023-01-17]
         YourPublicKey
   uid           [ultimate] yourUserName <yourEmail>
   sub   rsa2048 2021-01-17 [E] [expires: 2023-01-17]
   ```

4. 他人拿到了你部署之后的文件后, 需要进行校验, 校验过程需要用到你的公钥, 所以你需要将公钥上传到专门保存密钥的服务器, 以便他人
   可以获取公钥并验证签名 
    
    `gpg --keyserver hkp://pool.sks-keyservers.net --send-keys YourPublicKey`, 
   
    此处的公钥服务器还可以填写其他 server: `hkp://keys.gnupg.net`, `keyserver.ubuntu.com`
5. 可以使用 `gpg --keyserver hkp://pool.sks-keyservers.net --recv-keys YourPublicKey` 查看已经上传成功的公钥
6. 在项目的 pom.xml 文件中也需要引入 `maven-gpg-plugin` 插件, 在每次打包过程中将文件签名, 可参考
   [此项目的pom.xml](https://github.com/bigwolftime/redefine/blob/main/pom.xml).

#### 六. 部署项目

执行 deploy 命令: `mvn deploy -P signing-deploy`, 同时对 deploy 的文件签名, 执行完毕后, 
打开 [Nexus Repo Manager](https://oss.sonatype.org/) 并登录账号后, 点击 `stagingRepositories`, 可以看到刚刚部署的项目, 我们需要
将其改为 `Release` 状态, 不过在此之前需要验证 `pom.xml` 的配置(ex: 例如: name, description, license, scm, developers etc.)以及签名等项目. 

如何验证呢? 点击 `close` 后会自动开始验证, 可以在下方的 `Activity` 标签看到校验项及结果, 出现错误后会自动停止, 我们需要修复后重新部署,
直到提示: close successful, 之后再点击 `Release` 按钮可以将状态变更为 `Release`.

> 如下图所示, 已经成功 close, 并且通过了校验, 此时 Release 按钮变为可点击状态。

![Staging](https://zonheng.net/tech/67113191.png-original)

完成这一步, 需要回到 sonatype 的 issue 下, 通知工作人员已完成.

审核通过之后就会收到回复:

Central sync is activated for yourGroupId. After you successfully release, your component will be published to Central, 
typically within 10 minutes, though updates to search.maven.org can take up to two hours.

1-2h 之后, 即可在 [https://search.maven.org](https://search.maven.org) 搜索到你的项目.

今后的更新, 需要重复进行此步骤, 即 `mvn deploy - validate(close) - release` 过程


**完整的项目代码可以参考: [redefine](https://github.com/bigwolftime/redefine)**

#### 其他

1. 项目 License

   项目的许可证也是一个很重要的问题, 在为你的项目选择许可证之前, 必须仔细阅读并理解, 并知晓如何正确使用. 
   
   关于如何选择 License 可以参考: [Choose an open source license](https://choosealicense.com/)

2. 签名过程中出现 `mvn deploy occur: gpg: signing failed: Inappropriate ioctl for device`

   可参考[此文](https://my.oschina.net/ujjboy/blog/3023151), 原因是在 deploy 过程中需要输入生成密钥时的密码, 但当前可能不支持.
   
   在 shell 中执行: `export GPG_TTY=$(tty)`

3. 提示: `gpg2: command not found`

[参考此处](https://stackoverflow.com/questions/54391696/gpg2-command-not-found-even-when-gpg2-is-installed-on-mac-trying-to-install-rv)


#### 参考

1. [OSSRH guide](https://central.sonatype.org/pages/ossrh-guide.html)
2. [Sufficient Metadata](https://central.sonatype.org/pages/requirements.html)
3. [PGP signatures guide](https://central.sonatype.org/pages/working-with-pgp-signatures.html)
4. [如何将 Apache License 2.0 应用到你的项目](http://adoyle.me/blog/how-to-apply-the-apache-2-0-license-to-your-project.html)
5. [An Example - Maven Repository Format](https://help.sonatype.com/repomanager3/repository-manager-concepts/an-example---maven-repository-format)