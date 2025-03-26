---
title: 依赖管理之BOM
slug: using-maven-bill-of-materials-bom
date: 2024-08-31
desc: 本文详细介绍了BOM（材料清单）的概念、格式以及如何在项目中使用。BOM实际上是一个特殊的POM文件，它列出了一个工程的所有依赖及其对应版本，便于其他工程引用而无需指定具体版本。文章通过一个具体的BOM文件示例，解释了其关键信息和结构，包括打包方式、依赖管理等。
featuredImage: https://sonder.vitah.me/featured/3017adaff16219873d7d1625c4ab948f.webp
categories:
  - Tools
tags:
  - Java
---

## 问题

1. 什么是 BOM
2. BOM格式是什么
3. 怎么在项目中使用BOM

## 什么是 BOM

BOM 全称是 Bill Of Materials，译作材料清单。BOM 本身并不是一种特殊的文件格式，而是一个普通的 pom 文件，只是在这个 POM 中，我们罗列的是一个工程的所有依赖和其对应的版本。该文件一般被其它工程使用，当其它工程引用 BOM 中罗列的 jar 包时，不用显示指定具体的版本，会自动使用 BOM 对应的 jar 版本。

所以 BOM 的好处是用来管理一个工程的所有依赖版本信息。

## BOM 的格式

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
         xmlns="<http://maven.apache.org/POM/4.0.0>"
         xsi:schemaLocation="<http://maven.apache.org/POM/4.0.0> <http://maven.apache.org/xsd/maven-4.0.0.xsd>">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.vitahlin</groupId>
    <artifactId>janna-bom</artifactId>
    <!--BOM清单中不能使用revision-->
    <version>${bom.version}</version>
    <packaging>pom</packaging>

    <properties>        
        <bom.version>1.0-SNAPSHOT</bom.version>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <encoding>UTF-8</encoding>

        <lombok.version>1.18.26</lombok.version>
    </properties>

    <dependencyManagement>        
        <dependencies>
            <dependency>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>${lombok.version}</version>
                <scope>provided</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

</project>
```

其中的关键信息是：
- `<packaging>pom</packaging>` 打包方式是 pom 文件
- `<dependencyManagement><dependencies>` 下定义的各种依赖的版本

## 怎么引用 BOM

一般情况下，是在项目主 pom 中引入 BOM，比如 parent 的 pom 文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
         xmlns="<http://maven.apache.org/POM/4.0.0>"
         xsi:schemaLocation="<http://maven.apache.org/POM/4.0.0> <http://maven.apache.org/xsd/maven-4.0.0.xsd>">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.vitahlin</groupId>
    <artifactId>janna</artifactId>
    <version>${revision}</version>
    <packaging>pom</packaging>
    <modules>        
        <module>janna-basis</module>
        <module>janna-bom</module>
    </modules>

    <properties>        
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <revision>1.0-SNAPSHOT</revision>
        <bom.version>1.0-SNAPSHOT</bom.version>
    </properties>

    <dependencyManagement>        
        <dependencies>
            <dependency>
                <groupId>com.vitahlin</groupId>
                <artifactId>janna-bom</artifactId>
                <version>${bom.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

</project>
```

在需要使用相关 JAR 包的 pom.xml 文件中 `<dependencies></dependencies>` 节点下引入依赖的 groupId 和 artifactId 即可，如：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
         xmlns="<http://maven.apache.org/POM/4.0.0>"
         xsi:schemaLocation="<http://maven.apache.org/POM/4.0.0> <http://maven.apache.org/xsd/maven-4.0.0.xsd>">
    <modelVersion>4.0.0</modelVersion>
    <parent>        
        <groupId>com.vitahlin</groupId>
        <artifactId>janna</artifactId>
        <version>${revision}</version>
    </parent>
    <artifactId>janna-basis</artifactId>

    <properties>        
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
    </dependencies>
</project>
```

这时候，就能发现 janna-basis 模块直接引入了 lombok 依赖，并且版本就是在 janna-bom 中定义的版本：

![](https://vitahlin.oss-cn-shanghai.aliyuncs.com/sonder/2024/202403312254705.png)

**这样设置后，如果项目要求升级 lombok 的版本，只需要在提供方升级验证兼容性，然后修改 BOM 依赖即可**

如果需要使用不同于当前 bom 中所维护的 jar 包版本，则加上 `<version>` 覆盖即可，如：

```xml
<dependencies>  
    <dependency>  
        <groupId>org.projectlombok</groupId>  
        <artifactId>lombok</artifactId>  
        <version>1.18.24</version>  
        <scope>provided</scope>  
    </dependency>  
</dependencies>
```

## 版本冲突时的一些规则

当出现版本冲突时，具体使用哪一个版本的优先顺序是：
1. 直接在当前工程中显示指定的版本
2. parent 中配置的父工程使用的版本
3. 在当前工程中通过 `dependencyManagement` 引入的 BOM 清单中的版本，当引入的多个 BOM 都有对应 jar 包时，先引入的 BOM 生效
4. 上述三个地方都没配置，则启用依赖调解 dependency mediation

什么是依赖调解？
**Dependency mediation**（依赖调解）是指在多模块或多组件系统中，如何选择和解决不同版本的依赖关系。当项目或应用程序中引用了多个依赖，而这些依赖之间可能有版本冲突时，依赖调解的作用就是决定最终使用哪一个版本的依赖，以保证系统的稳定性和兼容性。

Maven 在面对版本冲突时，会选择“最近的依赖”（即根据依赖树的深度来选择），或者通过指定明确的版本来解决。
当有两个依赖路径，依赖到同一个 jar 的不同版本时，最短路径的版本生效。假设有如下依赖关系：
```shell
A -> B -> C -> D 1.4
A -> E -> D 1.0
```

在 Maven 中，最终会选择 D 的 **1.4 版本**，而不是 1.0 版本，因为 D 1.4 是通过路径 A -> B -> C -> D 1.4 被依赖的，而这个路径比 A -> E -> D 1.0 更长，所以 Maven 会选择版本较新的 D 1.4。