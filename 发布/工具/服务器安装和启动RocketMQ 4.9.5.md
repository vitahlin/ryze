---
slug: install-and-run-rocketmq4-9-5
title: 服务器安装和启动RocketMQ 4.9.5
date: 2024-06-25
desc: 这篇博客介绍了如何在服务器上安装和启动 RocketMQ 4.9.5。文章详细描述了 JDK 的安装过程，RocketMQ 4.9.5 的下载和解压步骤，以及如何调整运行环境配置以适应不同内存的服务器。还涵盖了启动 nameserver 和 broker 服务的具体操作，并通过命令行进行简单测试。
featuredImage: https://sonder.vitah.me/featured/f2cb215ea64a1de39882173203b45b4f.webp
tags:
  - RocketMQ
categories:
  - MessageQueue
---

## 安装JDK

首先需要安装 sdkman，参考官网教程：[Installation - SDKMAN! the Software Development Kit Manager](https://sdkman.io/install)


安装完后使用 sdkman 来安装 jdk：

```bash
 sdk install java 8.0.402-zulu
```


## 安装4.9.5

当前最新的版本是5.x，这是一个着眼于云原生的新版本，给 RocketMQ 带来了非常多很亮眼的新特性。但是目前来看，企业中用得还比较少。因此，我们这里采用的还是更为稳定的4.9.5版本。

```bash
wget <https://archive.apache.org/dist/rocketmq/4.9.5/rocketmq-all-4.9.5-bin-release.zip>
unzip rocketmq-all-4.9.5-bin-release.zip
```

解压后目录如下：

```bash
.
|-- benchmark
|-- bin
|-- conf
|-- lib
|-- LICENSE
|-- NOTICE
`-- README.md
```

- benchmark 压测脚本
- bin 执行脚本
- conf 配置文件
- lib 运行的jar包

## 运行

### 修改运行环境配置

RocketMQ 建议的运行环境需要至少12G的内存，这是生产环境比较理想的资源配置。但是，学习阶段，如果你的服务器没有这么大的内存空间，那么就需要做一下调整。进入 `bin` 目录，对其中的 `runserver.sh` 和 `runbroker.sh` 两个脚本进行修改。

修改 `bin/runserver.sh` ，找到

```bash
JAVA_OPT="${JAVA_OPT} -server -Xms4g -Xmx4g -Xmn2g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
```

调整Java进程的内存大小：

```bash
JAVA_OPT="${JAVA_OPT} -server -Xms512m -Xmx512m -Xmn256g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
```

修改 `bin/runbroker.sh` ，找到

```bash
JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g"
```

修改为

```bash
JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g"
```

调整完成后，就可以启动 RocketMQ 服务了。

**注：生产环境不建议调整**


### 启动nameserver服务


显示启动 `bin/mqnamesrv` ，运行结果如下：

```bash
OpenJDK 64-Bit Server VM warning: Using the DefNew young collector with the CMS collector is deprecated and will likely be removed in a future release
OpenJDK 64-Bit Server VM warning: UseCMSCompactAtFullCollection is deprecated and will likely be removed in a future release.
The Name Server boot success. serializeType=JSON
```

后台启动 `nohup bin/mqnamesrv &` 然后可以执行 `jsp` 命令可以看到有一个 NamesrvStartup 的进程运行：

```bash
319013 Jps
318994 NamesrvStartup
```

### 启动 broker 服务

启动broker服务前，为了方便我们测试，需要修改 `conf/broker.conf` 文件，追加配置：

```bash
autoCreateTopicEnable=true
```

表示允许 broker 端自动创建新的 Topic.

> 另外，如果服务器配置了多张网卡，比如阿里云，腾讯云这样的云服务器，他们通常有内网网卡和外网网卡两张网卡，那么需要增加配置 `brokerIP1`  属性，指向服务器的外网IP 地址，这样才能确保从其他服务器上访问到 RocketMQ 服务。

如：
```bash
brokerIP1=xxx.xxx.xxx.xxx
autoCreateTopicEnable=true
```

broker 不会读取默认的配置文件，需要指定配置文件：
```bash
bin/mqbroker -c conf/broker.conf
```

命令和 nameserver 类似，启动成功后会看到如下日志：
```bash
The broker[iZ2ze87wqglz1vylrsw0m1Z, xxx.xxx.xxx.xxx:10911] boot success. serializeType=JSON and name server is localhost:9876
```

注意如果日志内容只有：
```bash
The broker[iZ2ze87wqglz1vylrsw0m1Z, xxx.xxx.xxx.xxx:10911] boot success. serializeType=JSON
```

那表示 broker 没有正确连接到 nameserver，这时候一般需要修改防火墙配置。

#### 查看 broker 的配置参数

命令：

```bash
sh bin/mqbroker -m
```

运行结果
```bash
2024-04-16 20\\:32\\:38 INFO main - namesrvAddr=localhost:9876
2024-04-16 20\\:32\\:38 INFO main - brokerIP1=172.20.95.70
2024-04-16 20\\:32\\:38 INFO main - brokerName=iZ2ze87wqglz1vylrsw0m1Z
2024-04-16 20\\:32\\:38 INFO main - brokerClusterName=DefaultCluster
2024-04-16 20\\:32\\:38 INFO main - brokerId=0
2024-04-16 20\\:32\\:38 INFO main - autoCreateTopicEnable=true
2024-04-16 20\\:32\\:38 INFO main - autoCreateSubscriptionGroup=true
2024-04-16 20\\:32\\:38 INFO main - msgTraceTopicName=RMQ_SYS_TRACE_TOPIC
2024-04-16 20\\:32\\:38 INFO main - traceTopicEnable=false
2024-04-16 20\\:32\\:38 INFO main - rejectTransactionMessage=false
2024-04-16 20\\:32\\:38 INFO main - fetchNamesrvAddrByAddressServer=false
2024-04-16 20\\:32\\:38 INFO main - transactionTimeOut=6000
2024-04-16 20\\:32\\:38 INFO main - transactionCheckMax=15
2024-04-16 20\\:32\\:38 INFO main - transactionCheckInterval=60000
2024-04-16 20\\:32\\:38 INFO main - aclEnable=false
2024-04-16 20\\:32\\:38 INFO main - storePathRootDir=/root/store
2024-04-16 20\\:32\\:38 INFO main - storePathCommitLog=
2024-04-16 20\\:32\\:38 INFO main - storePathDLedgerCommitLog=
2024-04-16 20\\:32\\:38 INFO main - flushIntervalCommitLog=500
2024-04-16 20\\:32\\:38 INFO main - commitIntervalCommitLog=200
2024-04-16 20\\:32\\:38 INFO main - flushCommitLogTimed=true
2024-04-16 20\\:32\\:38 INFO main - deleteWhen=04
2024-04-16 20\\:32\\:38 INFO main - fileReservedTime=72
2024-04-16 20\\:32\\:38 INFO main - maxTransferBytesOnMessageInMemory=262144
2024-04-16 20\\:32\\:38 INFO main - maxTransferCountOnMessageInMemory=32
2024-04-16 20\\:32\\:38 INFO main - maxTransferBytesOnMessageInDisk=65536
2024-04-16 20\\:32\\:38 INFO main - maxTransferCountOnMessageInDisk=8
2024-04-16 20\\:32\\:38 INFO main - accessMessageInMemoryMaxRatio=40
2024-04-16 20\\:32\\:38 INFO main - messageIndexEnable=true
2024-04-16 20\\:32\\:38 INFO main - messageIndexSafe=false
2024-04-16 20\\:32\\:38 INFO main - haMasterAddress=
2024-04-16 20\\:32\\:38 INFO main - brokerRole=ASYNC_MASTER
2024-04-16 20\\:32\\:38 INFO main - flushDiskType=ASYNC_FLUSH
2024-04-16 20\\:32\\:38 INFO main - cleanFileForciblyEnable=true
2024-04-16 20\\:32\\:38 INFO main - transientStorePoolEnable=false
[1]+  Exit 137                nohup bin/mqbroker
```


## 测试

执行脚本进行测试：
```bash
export NAMESRV_ADDR=localhost:9876
sh tools.sh org.apache.rocketmq.example.quickstart.Producer
```

这个指令会默认往 RocketMQ 中发送1000条消息。在命令行窗口可以看到发送消息的日志。运行结果如下 ：
```bash
SendResult [sendStatus=SEND_OK, msgId=7F000001E29F2FF4ACD051AC3BAF0000, offsetMsgId=AC145F4600002A9F0000000000000000, messageQueue=MessageQueue [topic=TopicTest, brokerName=iZ2ze87wqglz1vylrsw0m1Z, queueId=2], queueOffset=0]
SendResult [sendStatus=SEND_OK, msgId=7F000001E29F2FF4ACD051AC3BF30001, offsetMsgId=AC145F4600002A9F00000000000000BE, messageQueue=MessageQueue [topic=TopicTest, brokerName=iZ2ze87wqglz1vylrsw0m1Z, queueId=3], queueOffset=0]
SendResult [sendStatus=SEND_OK, msgId=7F000001E29F2FF4ACD051AC3BF60002, offsetMsgId=AC145F4600002A9F000000000000017C, messageQueue=MessageQueue [topic=TopicTest, brokerName=iZ2ze87wqglz1vylrsw0m1Z, queueId=0], queueOffset=0]
SendResult [sendStatus=SEND_OK, msgId=7F000001E29F2FF4ACD051AC3C000003, offsetMsgId=AC145F4600002A9F000000000000023A, messageQueue=MessageQueue [topic=TopicTest, brokerName=iZ2ze87wqglz1vylrsw0m1Z, queueId=1], queueOffset=0]
```


可以启动消费者接收之前发送的消息：

```bash
sh tools.sh org.apache.rocketmq.example.quickstart.Consumer
```


运行结果如下：

```bash
ConsumeMessageThread_please_rename_unique_group_name_4_16 Receive New Messages: [MessageExt [brokerName=broker-a, queueId=0, storeSize=192, queueOffset=451, sysFlag=0, bornTimestamp=1713280030517, bornHost=/123.56.164.113:39870, storeTimestamp=1713280030519, storeHost=/123.56.164.113:10911, msgId=7B38A47100002A9F0000000000054976, commitLogOffset=346486, bodyCRC=1271400251, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=509, CONSUME_START_TIME=1713280077228, UNIQ_KEY=7F00000113132FF4ACD0523563350302, CLUSTER=DefaultCluster, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 32, 55, 55, 48], transactionId='null'}]]
ConsumeMessageThread_please_rename_unique_group_name_4_9 Receive New Messages: [MessageExt [brokerName=broker-a, queueId=0, storeSize=192, queueOffset=450, sysFlag=0, bornTimestamp=1713280030472, bornHost=/123.56.164.113:39870, storeTimestamp=1713280030475, storeHost=/123.56.164.113:10911, msgId=7B38A47100002A9F0000000000054676, commitLogOffset=345718, bodyCRC=1001427791, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=509, CONSUME_START_TIME=1713280077228, UNIQ_KEY=7F00000113132FF4ACD05235630802FE, CLUSTER=DefaultCluster, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 32, 55, 54, 54], transactionId='null'}]] 
```
