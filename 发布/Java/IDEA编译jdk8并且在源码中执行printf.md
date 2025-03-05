---
title: IDEA编译jdk8并且在源码中执行printf
slug: idea-compile-jdk8-execute-printf-in-source
date: 2024-02-10
categories:
  - Source Code
tags:
  - Java
featuredImage: https://sonder.vitah.me/featured/ab23011dab6256edafb54a3ca3e42a62.webp
desc: 介绍如何调试JDK源码的详细教程，方便我们更好的深入理解Java语言和JVM的内部机制。
---

## 从本机的jdk目录下载源码

如图所示：

![](https://sonder.vitah.me/2024/7eab377ed657b4f9d6957391f8fcdf65.webp)

打开任何一个 JDK8 项目查看 JDK8源码目录，这里 `JDK home path` 即 JDK 的安装目录，跳转到 JDK8 的这个目录路径：`/Users/vitah/.sdkman/candidates/java/8.0.312-tem`。

```shell
> cd /Users/vitah/.sdkman/candidates/java/8.0.312-tem
> tree -L 1
.
├── ASSEMBLY_EXCEPTION
├── LICENSE
├── NOTICE
├── THIRD_PARTY_README
├── bin
├── bundle
├── include
├── jre
├── lib
├── man
├── release
├── sample
└── src.zip

7 directories, 6 files
```

可以看到目录下有一个 `src.zip` 的压缩包，这个即是 `JDK8` 的源代码压缩包。

## 新建项目导入源代码

新建名为 `jdk8-analyse` 的空目录，将 `src.zip` 解压代码导入当前文件夹，结构如下：

```shell
jdk8-analyse
└── src
    ├── com
    ├── java
    ├── javax
    ├── jdk
    ├── launcher
    ├── org
    └── sun

8 directories
```

导入后，直接用 `Intellij-Idea` 打开项目 `jdk8-analyse`。

### 编译错误处理

#### java: Compilation failed: internal java compiler error

编译时，出现如下错误：

```shell
java: Compilation failed: internal java compiler error
```

调整 Idea 配置：`Preferences` → `Build, Execution, Deployment` → `Compiler`，修改 `Shared build process heap size` 的大小，将700改为3000。

#### snmp 相关类缺失

提示 `SnmpOid` 相关类缺失，具体错误如下：

```shell
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/com/sun/jmx/snmp/daemon/SnmpAdaptorServer.java:53:24
java: cannot find symbol
  symbol:   class SnmpOid
  location: package com.sun.jmx.snmp
  
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/com/sun/jmx/snmp/daemon/SnmpAdaptorServer.java:55:24
java: cannot find symbol
  symbol:   class SnmpPduPacket
  location: package com.sun.jmx.snmp
  
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/com/sun/jmx/snmp/daemon/SnmpAdaptorServer.java:58:24
java: cannot find symbol
  symbol:   class SnmpTimeticks
  location: package com.sun.jmx.snmp
  
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/com/sun/jmx/snmp/daemon/SnmpAdaptorServer.java:59:24
java: cannot find symbol
  symbol:   class SnmpVarBind
  location: package com.sun.jmx.snmp
```

这是因为 OpenJDK 采用 GPL V2协议，Oracle JDK 采用 JRL 协议，两者虽然都是开放源代码的，但是 GPL V2 允许商业使用，而 JRL 只允许个人研究使用。SUN JDK 的一部分源代码因为产权的问题无法开放给 OpenJDK 使用，其中最主要的部分就是 `jmx` 中的可选元件 `snmp` 部分的代码。

解决方案：下载 `snmp` 相关缺失类，直接复制到对应目录中。

#### DuctusRenderingEngine类相关包缺失

具体错误内容：

```shell
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/sun/dc/DuctusRenderingEngine.java:39:17
java: package sun.dc.pr does not exist
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/sun/dc/DuctusRenderingEngine.java:40:17
java: package sun.dc.pr does not exist
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/sun/dc/DuctusRenderingEngine.java:41:17
java: package sun.dc.pr does not exist
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/sun/dc/DuctusRenderingEngine.java:42:17
java: package sun.dc.pr does not exist
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/sun/dc/DuctusRenderingEngine.java:43:19
java: package sun.dc.path does not exist
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/sun/dc/DuctusRenderingEngine.java:44:19
java: package sun.dc.path does not exist
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/sun/dc/DuctusRenderingEngine.java:45:19
java: package sun.dc.path does not exist
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/sun/dc/DuctusRenderingEngine.java:174:54
java: cannot find symbol
  symbol:   class PathConsumer
  location: class sun.dc.DuctusRenderingEngine
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/sun/dc/DuctusRenderingEngine.java:176:16
java: cannot find symbol
  symbol:   class PathException
  location: class sun.dc.DuctusRenderingEngine
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/sun/dc/DuctusRenderingEngine.java:360:20
java: cannot find symbol
  symbol:   class Rasterizer
  location: class sun.dc.DuctusRenderingEngine
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/sun/dc/DuctusRenderingEngine.java:362:32
java: cannot find symbol
  symbol:   class Rasterizer
  location: class sun.dc.DuctusRenderingEngine
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/sun/dc/DuctusRenderingEngine.java:372:52
java: cannot find symbol
  symbol:   class Rasterizer
  location: class sun.dc.DuctusRenderingEngine
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/sun/dc/DuctusRenderingEngine.java:720:31
java: cannot find symbol
  symbol:   class PathConsumer
  location: class sun.dc.DuctusRenderingEngine
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/sun/dc/DuctusRenderingEngine.java:770:42
java: cannot find symbol
  symbol:   class PathConsumer
  location: class sun.dc.DuctusRenderingEngine
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/sun/dc/DuctusRenderingEngine.java:787:16
java: cannot find symbol
  symbol:   class PathConsumer
  location: class sun.dc.DuctusRenderingEngine.FillAdapter
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/sun/dc/DuctusRenderingEngine.java:826:30
java: cannot find symbol
  symbol:   class FastPathProducer
  location: class sun.dc.DuctusRenderingEngine.FillAdapter
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/sun/dc/DuctusRenderingEngine.java:827:20
java: cannot find symbol
  symbol:   class PathException
  location: class sun.dc.DuctusRenderingEngine.FillAdapter
```

这里直接删除 `sun/dc/DuctusRenderingEngine` 类 。

#### sun.nio.fs包下相关类缺失

具体错误内容：

```shell
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/sun/nio/fs/SolarisUserDefinedFileAttributeView.java:36:25
java: cannot find symbol
  symbol:   class SolarisConstants
  location: package sun.nio.fs
```

解决方式：直接删除 `sun.nio.fs` 整个包。

#### XEvent类缺失

具体错误：

```shell
/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src/sun/awt/X11/XComponentPeer.java:1242:39
java: cannot find symbol
  symbol:   class XEvent
  location: class sun.awt.X11.XComponentPeer
```

解决方式：直接删除 `sun/awt/X11` 包。

## 调试jdk源码，自由注释修改

### 将JDK换成当前项目

新建测试类 `ArrayListTest`：

```java
public class ArrayListTest {
    public static void main(String[] args) {
        List list = new ArrayList();
        list.add(1);
        System.out.println(list.size());
    }
}
```

跳转到 `ArrayList` 的构造函数，发现 `ArrayList.java` 文件还是只读的，这是因为现在看到的 `ArrayList.java` 代码还是本机安装的 `jdk8`，而不是刚才导入的源代码。

`Idea` → `File` → `Project Structrue`，如图所示：

![](https://sonder.vitah.me/2024/1e3b15d279f2c5d07774191300411489.webp)

可以看到，上述图中使用的 jdk 源代码仍然是系统安装的。这里将其切换为当前项目的 jdk 代码。

选择新增 SDK，`JDK home path` 和原来的保持一致，这里配置成 `/Users/vitah/.sdkman/candidates/java/8.0.312-tem`，修改 `Sourcepath`，选择当前项目的 src 目录，这里选择 `/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/src`：

![](https://sonder.vitah.me/2024/9308c5c64f29fe5c453cd5c4527f95b6.webp)

点击 `Apply` 保存，然后修改当前项目的 `sdk` 为 `java8-analyse`，新建测试类 `ArrayListTest` ：

```java
public class ArrayListTest {
    public static void main(String[] args) {
        List list = new ArrayList();
        list.add(1);
        System.out.println(list.size());
    }
}
```

并且修改，`ArrayList` 的构造函数如下图：

![](https://sonder.vitah.me/2024/d51001e3fb7ad0eaf2c6de75909fe31c.webp)

执行输出结果如下：

```shell
/Users/vitah/.sdkman/candidates/java/8.0.312-tem/bin/java -javaagent:/Applications/IntelliJ IDEA.app/Contents/lib/idea_rt.jar=58732:/Applications/IntelliJ IDEA.app/Contents/bin -Dfile.encoding=UTF-8 -classpath /Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/charsets.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/ext/cldrdata.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/ext/dnsns.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/ext/jaccess.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/ext/localedata.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/ext/nashorn.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/ext/sunec.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/ext/sunjce_provider.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/ext/sunpkcs11.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/ext/zipfs.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/jce.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/jfr.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/jsse.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/management-agent.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/resources.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/rt.jar:/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/out/production/jdk8-analyse vitahlin.ArrayListTest
1

Process finished with exit code 0
```

发现在构造函数里面的 `println` 函数并不能打印。

### 使jdk中自己修改的代码能够执行

如何使 `jdk` 代码中的 `println` 函数生效，可以参考这个链接：[https://www.cnblogs.com/grey-wolf/p/12817615.html#_label4](https://www.cnblogs.com/grey-wolf/p/12817615.html#_label4)

这个值的生成规则可以参考代码（运行时，修改 `myPath` 为自己本机路径即可）：

```java
public static void main(String[] args) {
    String pathTotal = System.getProperty("sun.boot.class.path");
    String[] paths = pathTotal.split(":");
    Arrays.stream(paths).forEach(System.out::println);

    String myPath = "/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/out/production/jdk8-analyse";
    String vmOption = "-Dsun.boot.class.path=" + myPath + ":" + pathTotal;
    System.out.println("\nCopy the result and add to vm option:");
    System.out.println(vmOption);
}
```

执行结果为：

```shell
/Users/vitah/.sdkman/candidates/java/8.0.312-tem/bin/java -javaagent:/Applications/IntelliJ IDEA.app/Contents/lib/idea_rt.jar=59625:/Applications/IntelliJ IDEA.app/Contents/bin -Dfile.encoding=UTF-8 -classpath /Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/charsets.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/ext/cldrdata.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/ext/dnsns.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/ext/jaccess.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/ext/localedata.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/ext/nashorn.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/ext/sunec.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/ext/sunjce_provider.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/ext/sunpkcs11.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/ext/zipfs.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/jce.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/jfr.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/jsse.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/management-agent.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/resources.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/rt.jar:/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/out/production/jdk8-analyse vitahlin.GenerateVmOption
/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/resources.jar
/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/rt.jar
/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/sunrsasign.jar
/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/jsse.jar
/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/jce.jar
/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/charsets.jar
/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/jfr.jar
/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/classes

Copy the result and add to vm option:
-Dsun.boot.class.path=/Users/vitah/Downloads/dev/vitah/jarvan/jdk8-analyse/out/production/jdk8-analyse:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/resources.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/rt.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/sunrsasign.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/jsse.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/jce.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/charsets.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/lib/jfr.jar:/Users/vitah/.sdkman/candidates/java/8.0.312-tem/jre/classes

Process finished with exit code 0
```

**复制12行的内容将其配置为** `**ArrayListTest**` **的** `**vm option**`

然后再执行，会看到现在执行结果如下：

```shell
This is my ArrayList
This is my ArrayList
This is my ArrayList
This is my ArrayList
This is my ArrayList
This is my ArrayList
This is my ArrayList
This is my ArrayList
1

Process finished with exit code 0
```

这样，就可以给 `jdk` 自由增加注释修改源代码了。

## 参考链接

1. [https://my.oschina.net/u/2518341/blog/1931088](https://my.oschina.net/u/2518341/blog/1931088)
2. [https://www.cnblogs.com/grey-wolf/p/12817615.html](https://www.cnblogs.com/grey-wolf/p/12817615.html)
