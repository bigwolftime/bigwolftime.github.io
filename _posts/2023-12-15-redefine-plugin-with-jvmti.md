---
layout: post
title: Java 代码动态调试, 字节码增强技术简介及实践
date: 2024-12-15
Author: bwt
categories: JVM
tags: [Java, JVMTI, 字节码, Instrument]
comments: true
toc: true
---

本文简单介绍下 Java 虚拟机提供的 JVMTI 以及 Instrument, 以及如何利用 Instrument 技术开发一个 jar 插件, 实现代码的动态更新(热部署), 
进而提升开发效率.

<!--break-->

### 一. 前言

在日常的开发过程中, 你是否遇到过以下类似场景? 

一段代码的执行逻辑不正确, 想排查问题, 但关键的出入参日志没有打印, 想要排查无从下手, 只能在代码中补充上相关日志后, 重新发起编译&部署的流程.
待定位到问题后, 还要再次修改正代码, 重新发起编译&部署. 这样频繁的服务器重启/部署操作不但可能会影响到其他同学的测试与验证, 还会影响开发效率, 
假如你的项目比较臃肿, 则大部分的时间都可能花在了代码的编译和服务启动上.

又或者, 一段逻辑执行缓慢, 但不知道哪里的问题, 只能将内部的方法逐一加上执行时长埋点, 加上埋点后又需要经过编译&部署的流程...

甚至是一些低级错误: if 判断写反了, 出现空指针异常, 变量赋值错误等等, 这些都免不了编译和部署.

那么, 有什么方案能在一定程度上简化下上面的流程呢? 这就需要引入 Java 虚拟机提供的一项功能: `JVM TI(JVM Tool Interface)`, 及其对应的实现:
Instrument, 有了它, 不论是代码中加日志, 加埋点, 修复 if 判断, 还是处理空指针异常, 都可以实现快速编译(只编译修改的类)以及热部署.

> Instrument 和 JVMTI 分别是什么? 有什么关系吗?
> 
> * Instrumentation 建立在 JVMTI 基础之上, 利用 JVMTI 提供的 JVM 接口和事件回调, 来实现一套程序探测、分析和修改的框架。
> 
> * JVMTI 是原生的 Java 虚拟机接口,提供了内部访问、事件通知等底层功能。而 Instrumentation 则在 JVMTI 之上,构建了更丰富的应用程序代理和修改能力。
> 
> 因此, 两者是协同的关系: JVMTI 提供基础设施, Instrumentation 应用其实现扩展功能.


### 二. Instrument 功能简介

> `JVM TI(JVM Tool Interface)` 偏向底层, 理解较为晦涩, 便不多介绍, 有兴趣可以看参考部分的链接.

Instrumentation, 即 Java Instrumentation 技术,是 Java 提供的一套程序探测和修改的框架和API。它的主要特征和功能包括:

* 检测类的加载、卸载事件,也可以代理修改类的定义;
* 检测方法的调用进入和退出事件,可以收集或修改方法的参数、返回值;
* 设置断点来控制方法执行路径;
* 支持定义代理类来包装原有类的功能; 
* 实时计算Java程序的一些度量指标,如运行时间、空间等;
* 配合 JVMTI 可以访问、监控更丰富的虚拟机运行时信息.

有不少 APM 工具便是依赖了此技术, 例如: SkyWalking, Pinpoint, Elastic APM, BTrace, Arthas 等.

### 三. 实践

梳理下需求: 
* 以 jar 包的形式打包, 以供其他项目引入;
* 上传一份源代码, 服务器收到后可以进行动态编译;
* 编译完成之后, 利用 Instrument 提供的功能, 应用新的字节码;

#### 1. 初始化项目

新建一个 maven 项目. 既然要以 jar 形式打包, 那么需要对 pom.xml 进行一些配置:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.sonatype.oss</groupId>
        <artifactId>oss-parent</artifactId>
        <version>9</version>
    </parent>

    <!-- 需要改成你的 groupId -->
    <groupId>io.github.bigwolftime</groupId>
    <artifactId>redefine</artifactId>
    <packaging>jar</packaging>
    <version>0.0.1</version>

    <name>redefine</name>

    <licenses>
        <license>
            <name>Apache License, Version 2.0</name>
            <url>http://www.apache.org/licenses/LICENSE-2.0</url>
            <distribution>repo</distribution>
        </license>
    </licenses>

    <scm>
        <!-- 如果要将 jar 发布到 maven.org, 则需要将此处改成你的 github 地址 -->
        <url>https://github.com/bigwolftime/redefine.git</url>
        <connection>scm:git:git@github.com:bigwolftime/redefine.git</connection>
        <developerConnection>scm:git:git@github.com:bigwolftime/redefine.git</developerConnection>
        <tag>HEAD</tag>
    </scm>

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

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>

        <!-- Unify the encoding for all modules -->
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>

        <compiler.fork>1.8</compiler.fork>

        <!-- Plugins -->
        <maven-eclipse-plugin.version>2.10</maven-eclipse-plugin.version>
        <maven-idea-plugin.version>2.2.1</maven-idea-plugin.version>

        <maven-bundle-plugin.version>5.1.1</maven-bundle-plugin.version>
        <maven-assembly-plugin.version>3.3.0</maven-assembly-plugin.version>
        <maven-surefire-plugin.version>2.22.2</maven-surefire-plugin.version>
        <build-helper-maven-plugin.version>3.2.0</build-helper-maven-plugin.version>
        <maven-compiler-plugin.version>3.8.1</maven-compiler-plugin.version>
        <maven-javadoc-plugin.version>3.2.0</maven-javadoc-plugin.version>
        <maven-install-plugin.version>2.5.2</maven-install-plugin.version>
        <maven-scm-provider-gitexe.version>1.11.2</maven-scm-provider-gitexe.version>
        <nexus-staging-maven-plugin.version>1.6.8</nexus-staging-maven-plugin.version>
        <maven-source-plugin.version>3.2.1</maven-source-plugin.version>
        <maven-release-plugin.version>2.5.3</maven-release-plugin.version>
        <maven-deploy-plugin.version>2.8.2</maven-deploy-plugin.version>


    </properties>

    <dependencies>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-io</artifactId>
            <version>1.3.2</version>
        </dependency>
    </dependencies>

    <build>
        <defaultGoal>clean</defaultGoal>

        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-assembly-plugin</artifactId>
                    <version>${maven-assembly-plugin.version}</version>
                </plugin>

                <plugin>
                    <groupId>org.codehaus.mojo</groupId>
                    <artifactId>build-helper-maven-plugin</artifactId>
                    <version>${build-helper-maven-plugin.version}</version>
                </plugin>

                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>${maven-compiler-plugin.version}</version>
                    <configuration>
                        <source>${java.version}</source>
                        <target>${java.version}</target>
                        <fork>true</fork>
                    </configuration>
                </plugin>

                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-eclipse-plugin</artifactId>
                    <version>${maven-eclipse-plugin.version}</version>
                    <configuration>
                        <downloadSources>true</downloadSources>
                        <downloadJavadocs>false</downloadJavadocs>
                    </configuration>
                </plugin>

                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-idea-plugin</artifactId>
                    <version>${maven-idea-plugin.version}</version>
                    <configuration>
                        <downloadSources>true</downloadSources>
                        <downloadJavadocs>false</downloadJavadocs>
                    </configuration>
                </plugin>

                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-javadoc-plugin</artifactId>
                    <version>${maven-javadoc-plugin.version}</version>
                    <configuration>
                        <attach>true</attach>
                        <source>${java.version}</source>
                        <quiet>true</quiet>
                        <detectOfflineLinks>false</detectOfflineLinks>
                        <encoding>${project.build.sourceEncoding}</encoding>
                    </configuration>
                </plugin>

                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-install-plugin</artifactId>
                    <version>${maven-install-plugin.version}</version>
                </plugin>

                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-surefire-plugin</artifactId>
                    <version>${maven-surefire-plugin.version}</version>
                </plugin>

                <plugin>
                    <groupId>org.apache.felix</groupId>
                    <artifactId>maven-bundle-plugin</artifactId>
                    <version>${maven-bundle-plugin.version}</version>
                </plugin>

                <plugin>
                    <groupId>org.sonatype.plugins</groupId>
                    <artifactId>nexus-staging-maven-plugin</artifactId>
                    <version>${nexus-staging-maven-plugin.version}</version>
                    <extensions>true</extensions>
                    <configuration>
                        <serverId>oss-repo</serverId>
                        <nexusUrl>https://oss.sonatype.org/</nexusUrl>
                        <autoReleaseAfterClose>true</autoReleaseAfterClose>
                    </configuration>
                </plugin>

                <!-- 如果要将 jar 发布到 maven.org, 则需要此插件将 jar 签名才行 -->
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-gpg-plugin</artifactId>
                    <version>1.5</version>
                    <executions>
                        <execution>
                            <phase>verify</phase>
                            <goals>
                                <goal>sign</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </pluginManagement>


        <plugins>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-source-plugin</artifactId>
                <version>${maven-source-plugin.version}</version>
                <executions>
                    <execution>
                        <id>attach-sources</id>
                        <goals>
                            <goal>jar</goal>
                            <goal>test-jar</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

            <plugin>
                <artifactId>maven-deploy-plugin</artifactId>
                <version>${maven-deploy-plugin.version}</version>
                <executions>
                    <execution>
                        <id>default-deploy</id>
                        <phase>deploy</phase>
                        <goals>
                            <goal>deploy</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-javadoc-plugin</artifactId>
                <executions>
                    <execution>
                        <id>attach-javadocs</id>
                        <goals>
                            <goal>jar</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-release-plugin</artifactId>
                <version>${maven-release-plugin.version}</version>
                <configuration>
                    <localCheckout>true</localCheckout>
                    <pushChanges>false</pushChanges>
                    <mavenExecutorId>forked-path</mavenExecutorId>
                </configuration>
                <dependencies>
                    <dependency>
                        <groupId>org.apache.maven.scm</groupId>
                        <artifactId>maven-scm-provider-gitexe</artifactId>
                        <version>${maven-scm-provider-gitexe.version}</version>
                    </dependency>
                </dependencies>
            </plugin>

            <plugin>
                <groupId>pl.project13.maven</groupId>
                <artifactId>git-commit-id-plugin</artifactId>
                <version>4.0.1</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>revision</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <verbose>false</verbose>
                    <dateFormat>yyyy-MM-dd'T'HH:mm:ssZ</dateFormat>
                    <generateGitPropertiesFile>true</generateGitPropertiesFile>
                    <generateGitPropertiesFilename>./redefine-git.properties</generateGitPropertiesFilename>
                    <excludeProperties>
                        <excludeProperty>git.branch</excludeProperty>
                        <excludeProperty>git.build.host</excludeProperty>
                        <excludeProperty>git.build.time</excludeProperty>
                        <excludeProperty>git.build.user.email</excludeProperty>
                        <excludeProperty>git.build.user.name</excludeProperty>
                        <excludeProperty>git.remote.origin.url</excludeProperty>
                        <excludeProperty>git.total.commit.count</excludeProperty>
                        <excludeProperty>git.commit.time</excludeProperty>
                        <excludeProperty>git.local.branch.ahead</excludeProperty>
                        <excludeProperty>git.local.branch.behind</excludeProperty>
                        <excludeProperty>git.tags</excludeProperty>
                    </excludeProperties>
                    <injectAllReactorProjects>true</injectAllReactorProjects>
                </configuration>
            </plugin>
        </plugins>
    </build>

    <repositories>
        <repository>
            <id>jitpack.io</id>
            <url>https://jitpack.io</url>
        </repository>
    </repositories>

    <profiles>
        <profile>
            <id>signing-deploy</id>
            <activation>
                <property>
                    <name>gpg.passphrase</name>
                </property>
            </activation>
            <build>
                <plugins>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-gpg-plugin</artifactId>
                        <executions>
                            <execution>
                                <id>sign-artifacts</id>
                                <phase>verify</phase>
                                <goals>
                                    <goal>sign</goal>
                                </goals>
                                <configuration>
                                    <gpgArguments>
                                        <argument>--pinentry-mode</argument>
                                        <argument>loopback</argument>
                                    </gpgArguments>
                                </configuration>
                            </execution>
                        </executions>
                    </plugin>
                </plugins>
            </build>
        </profile>

        <profile>
            <id>deploy</id>
            <activation>
                <jdk>[1.8,1.8]</jdk>
            </activation>
            <build>

                <plugins>
                    <plugin>
                        <artifactId>maven-source-plugin</artifactId>
                        <executions>
                            <execution>
                                <id>attach-source</id>
                                <goals>
                                    <goal>jar</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>

                    <plugin>
                        <artifactId>maven-javadoc-plugin</artifactId>
                        <executions>
                            <execution>
                                <id>attach-javadoc</id>
                                <goals>
                                    <goal>jar</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>
                </plugins>

            </build>
        </profile>
    </profiles>
</project>
```

#### 2. 接收源代码内容

服务一般部署在远程机器上, 如何接收新的源代码内容呢? 我这里使用 WebServlet, 通过 http 接口的方式传输, 且看代码.

首先引入 WebServlet 依赖:

```xml
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>3.0.1</version>
    <scope>provided</scope>
</dependency>
```

然后写 service 层逻辑:

```java
@WebServlet("/backend/redefine/base64")
@MultipartConfig
public class ExtendRedefineServlet extends HttpServlet {

    // 处理 Post 类型的请求
    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        resp.setCharacterEncoding("UTF-8");
        resp.setContentType("text/html;charset=UTF-8");
        
        /*
         * 此处定义两个参数:
         * class: 要修改的类的全限定类名;
         * base6_file: 源代码的 base64 编码. 直传源代码内容可能会有问题, 所以此处用了 base64.
         */
        String cls = req.getParameter("class");
        String file = req.getParameter("base64_file");

        // decode 出源代码内容
        byte[] decode = Base64.getDecoder().decode(file);
        ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(decode);

        byte[] bytes = compileByArthas(byteArrayInputStream, cls);
        try {
            transformer(cls, bytes);
            print(resp, "redefine success.");
        } catch (ClassNotFoundException | UnmodifiableClassException e) {
            print(resp, "transform error. " + e.getMessage());
        }
    }
}

```

上面的代码逻辑不难, 就是获取到 class(全限定类名) 和 bytes(源代码内容的 byte 格式), 然后调用 compileByArthas 方法编译成字节码(此方法实现
参考了 Arthas), 并调用 transformer 方法替换, 替换完成后输出 `redefine success`.

#### 3. 动态编译

下面编写 compileByArthas 方法的详细逻辑:

```java
private byte[] compileByArthas(InputStream inputStream, String className) throws IOException {
    ClassLoader classLoader = ClassLoaderUtil.getClassLoaderByName(INSTRUMENT, className);
    if (Objects.isNull(classLoader)) {
        // default system class loader
        String jarPath = DynamicCompiler.class.getProtectionDomain().getCodeSource().getLocation().getFile();
        File file = new File(jarPath);
        classLoader = new URLClassLoader(new URL[] { file.toURI().toURL() }, ClassLoader.getSystemClassLoader().getParent());
    }

    DynamicCompiler dynamicCompiler = new DynamicCompiler(classLoader);

    String code = IOUtils.toString(inputStream, StandardCharsets.UTF_8.name());
    dynamicCompiler.addSource(className, code);

    Map<String, byte[]> byteCodes = dynamicCompiler.buildByteCodes();

    Iterator<byte[]> iterator = byteCodes.values().stream().iterator();
    return iterator.next();
}
```

首先, 要获取到加载该类的 classLoader, 即: `ClassLoaderUtil.getClassLoaderByName` 方法. 此方法的第一个入参 INSTRUMENT 从哪来? 
需要额外引入 ByteBuddy 帮我们获取.

```xml
<dependency>
    <groupId>net.bytebuddy</groupId>
    <artifactId>byte-buddy-agent</artifactId>
    <version>1.10.18</version>
</dependency>
```

引入 ByteBuddy 后在 service 中定义:

```java
private static final Instrumentation INSTRUMENT;

static {
    // 懒汉式加载
    INSTRUMENT = ByteBuddyAgent.install();
}
```

> 为何需要 INSTRUMENT 实例? 因为要通过 `INSTRUMENT.getAllLoadedClasses()` 方法找到所有的 classLoader, 再进而找到加载要替换的类的 
> classLoader. 此步骤相当重要, classLoader 一旦找错了, 动态编译时可能会报错: `Can not find symbol`, 即找不到依赖的类.

`ClassLoaderUtil.getClassLoaderByName` 的代码:

```java
public static ClassLoader getClassLoaderByName(Instrumentation instrumentation, String className) {
    Class[] classes = instrumentation.getAllLoadedClasses();
    for (Class<?> clazz : classes) {
        if (clazz.getName().equals(className)) {
            return clazz.getClassLoader();
        }
    }

    return null;
}
```

classLoader 找到了, 下一步就可以发起编译了, 编译的逻辑依靠 DynamicCompiler.

```java
public class DynamicCompiler {

    private final JavaCompiler javaCompiler = ToolProvider.getSystemJavaCompiler();
    private final StandardJavaFileManager standardFileManager;
    private final List<String> options = new ArrayList<>();
    private final DynamicClassLoader dynamicClassLoader;

    private final Collection<JavaFileObject> compilationUnits = new ArrayList<JavaFileObject>();
    private final List<Diagnostic<? extends JavaFileObject>> errors = new ArrayList<Diagnostic<? extends JavaFileObject>>();
    private final List<Diagnostic<? extends JavaFileObject>> warnings = new ArrayList<Diagnostic<? extends JavaFileObject>>();

    public DynamicCompiler(ClassLoader classLoader) {
        if (javaCompiler == null) {
            throw new IllegalStateException(
                    "Can not load JavaCompiler from javax.tools.ToolProvider#getSystemJavaCompiler(),"
                            + " please confirm the application running in JDK not JRE.");
        }
        standardFileManager = javaCompiler.getStandardFileManager(null, null, null);

        options.add("-Xlint:unchecked");
        dynamicClassLoader = new DynamicClassLoader(classLoader);
    }

    public void addSource(String className, String source) {
        addSource(new StringSource(className, source));
    }

    public void addSource(JavaFileObject javaFileObject) {
        compilationUnits.add(javaFileObject);
    }

    public Map<String, byte[]> buildByteCodes() {

        errors.clear();
        warnings.clear();

        JavaFileManager fileManager = new DynamicJavaFileManager(standardFileManager, dynamicClassLoader);

        DiagnosticCollector<JavaFileObject> collector = new DiagnosticCollector<JavaFileObject>();
        JavaCompiler.CompilationTask task = javaCompiler.getTask(null, fileManager, collector, options, null,
                compilationUnits);

        try {

            if (!compilationUnits.isEmpty()) {
                boolean result = task.call();

                if (!result || collector.getDiagnostics().size() > 0) {

                    for (Diagnostic<? extends JavaFileObject> diagnostic : collector.getDiagnostics()) {
                        switch (diagnostic.getKind()) {
                            case NOTE:
                            case MANDATORY_WARNING:
                            case WARNING:
                                warnings.add(diagnostic);
                                break;
                            case OTHER:
                            case ERROR:
                            default:
                                errors.add(diagnostic);
                                break;
                        }

                    }

                    if (!errors.isEmpty()) {
                        throw new DynamicCompilerException("Compilation Error", errors);
                    }
                }
            }

            return dynamicClassLoader.getByteCodes();
        } catch (ClassFormatError e) {
            throw new DynamicCompilerException(e, errors);
        } finally {
            compilationUnits.clear();

        }

    }

}
```

### 参考

https://github.com/alibaba/arthas
https://tech.meituan.com/2019/09/05/java-bytecode-enhancement.html
https://tech.meituan.com/2019/11/07/java-dynamic-debugging-technology.html
