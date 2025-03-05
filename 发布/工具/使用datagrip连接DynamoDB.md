---
title: 使用datagrip连接DynamoDB
slug: connect-datagrip-to-dynamodb
date: 2025-02-27
desc: 本文将介绍如何在 DataGrip 中配置 DynamoDB 连接，包括所需的驱动、连接设置，以及常见问题的解决方案，帮助你更高效地管理 DynamoDB 数据。
featuredImage: https://sonder.vitah.me/featured/8353501ca7cc56e829dbd7484d850d33.webp
categories:
  - Tools
tags:
  - tools
---

# 配置Profile

**Profile** 是 AWS 访问凭证的**命名配置**，默认是 `default`，但可以创建多个。

配置文件存储在： `~/.aws/credentials` 

如果不存在就新建文件， 内容如下：
```sql
[default]
aws_access_key_id = abc
aws_secret_access_key = def
region = us-east-2

[us-profile]
aws_access_key_id = abc
aws_secret_access_key = def
region = us-east-2
```

# datagrip连接配置

Host是固定的格式，`dynamodb.{aws-region}.amazonaws.com` ， `aws-region` 是亚马逊AWS的区域，详细配置如下图：
![](https://sonder.vitah.me/ryze/9583d5a49d664385fc050e0846b43b4d.webp)

`Profile` 取的是文件 `credentials` 的配置，没配置默认取 `default`。
