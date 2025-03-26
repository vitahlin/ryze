---
title: TCP三次握手流程
slug: tcp-three-way-handshake
categories:
  - Lang
tags:
  - C
featuredImage: https://sonder.vitah.me/ryze/c844b739df170ffc3996e9e86f9483d2.webp
desc: TCP 三次握手是建立可靠连接的核心过程，它通过 SYN、SYN-ACK 和 ACK 消息的交互，确保客户端和服务器能够顺利建立通信通道。本文将通过 Wireshark 抓包，详细展示 TCP 三次握手的具体流程，解析每一步的作用和背后的原理，帮助你更好地理解这一关键网络机制。
draft: "false"
---

### 概述

TCP是主机对主机层的传输控制协议，提供可靠的连接服务，采用三次握手确认建立一个连接。
这里通过`Wireshark`对简单的时间字符串TCP传输程序进行抓包，来分析三次握手的内容，理解TCP建立可靠连接的流程。

### TCP报文一些内容介绍

- `SYN`：建立连接，synchronous
- `ACK`：确认，acknowledgement
- `PSH`：传送，push
- `FIN`：结束，finish
- `RST`：重置，reset
- `URG`：紧急，urgent
- `Sequence number`：顺序号码，用来跟踪该端发送的数据量
- `Acknowledge number`：确认号码，用来通知发送端接收数据成功

### 三次握手流程抓包分析

编写一个示例代码，让客户端从服务器获取当前的时间。

服务器代码：
```c
#include <time.h>  
#include "../lib/constant.h"  
#include "../lib/unp.h"  
  
int main(int argc, char **argv) {  
    struct sockaddr_in serv_addr, cli_addr;  
    socklen_t len;  
    char buff[MAX_SIZE];  
    time_t ticks;  
  
    // SOCK_STREAM套接字类型  
    int listen_fd = wrapSocket(AF_INET, SOCK_STREAM, 0);  
  
    bzero(&serv_addr, sizeof(serv_addr));  
  
    // 协议族，AF_INET表示IPv4协议  
    serv_addr.sin_family = AF_INET;  
  
    // 套接字端口  
    serv_addr.sin_port = htons(9876);  
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);  
  
    wrapBind(listen_fd, (const struct sockaddr *) &serv_addr, sizeof(serv_addr));  
  
    // 将套接字转换成一个监听套接字，这样来自客户端的外来连接就可以在该套接字上由内核接受  
    wrapListen(listen_fd, LISTENQ);  
  
    printf("time server running...\n");  
    for (;;) {  
        // 调用accept返回用于传输的socket的文件描述符  
        int conn_fd = wrapAccept(listen_fd, (struct sockaddr *) &cli_addr, &len);  
  
        printf("New client connect IP=%s, port=%d, conn_id=%d\n",  
               inet_ntop(AF_INET, &cli_addr.sin_addr, buff, sizeof(buff)),  
               ntohs(cli_addr.sin_port),  
               conn_fd);  
        ticks = time(NULL);  
  
        snprintf(buff, sizeof(buff), "%.24s\r\n", ctime(&ticks));  
        wrapWriten(conn_fd, buff, sizeof(buff));  
  
        wrapClose(conn_fd);  
    }  
}
```

客户端代码 ：
```c
#include "../lib/constant.h"  
#include "../lib/unp.h"  
  
int main(int argc, char **argv) {  
    char serv_ip[16];  
    int port;  
  
    // 网络上可用的时间服务器IP地址132.163.96.5，端口13  
    printf("Input IP address: ");  
    scanf("%s", serv_ip);  
    printf("Input port: ");  
    scanf("%d", &port);  
  
    // 创建一个套接字  
    int sock_fd = wrapSocket(AF_INET, SOCK_STREAM, 0);  
  
    // 套接字地址  
    struct sockaddr_in serv_address;  
    bzero(&serv_address, sizeof(serv_address));  
  
    // 协议族  
    serv_address.sin_family = AF_INET;  
    // 指定端口，时间服务器端口13  
    serv_address.sin_port = htons(port);  
  
    // 进行IP地址转换  
    wrapInetPton(AF_INET, serv_ip, &serv_address.sin_addr);  
  
    wrapConnect(sock_fd, (const struct sockaddr *) &serv_address, sizeof(serv_address));  
  
    ssize_t n;  
    char receive_line[MAX_SIZE + 1];  
  
    // 这里循环是因为在TCP中，不一定一次就能拿到全部返回内容  
    while ((n = read(sock_fd, receive_line, MAX_SIZE)) > 0) {  
        receive_line[n] = 0;  
        if (fputs(receive_line, stdout) == EOF) {  
            printf("fputs error");  
            exit(-1);  
        }  
    }  
  
    if (n < 0) {  
        printf("read error");  
        exit(-1);  
    }  
  
    return 0;  
}
```

代码实现的逻辑是启动服务器后，客户端连接服务器打印从服务器收到的时间字符串，然后客户端断掉连接。
接着，我们通过`Wireshark`来分析这整个流程。

抓包整个流程如下图：
![](https://sonder.vitah.me/ryze/0f80ac3a7314939c0d3f8ff49817ae4e.webp)

#### 第一次握手

如上图所示，选择 **序号为1** 的包，因为我们是要分析TCP报文，右键`Transmission Control Protocol`项，选择`Expand All`展开全部项，然后选择`Copy-All Visible Selected Tree Items`复制TCP报文中的全部内容，结果如下：

```c
Transmission Control Protocol, Src Port: 51567 (51567), Dst Port: sd (9876), Seq: 1536975978, Len: 0
    Source Port: 51567 (51567)
    Destination Port: sd (9876)
    [Stream index: 0]
    [TCP Segment Len: 0]
    Sequence number: 1536975978
    [Next sequence number: 1536975978]
    Acknowledgment number: 0
    1011 .... = Header Length: 44 bytes (11)
    Flags: 0x002 (SYN)
        000. .... .... = Reserved: Not set
        ...0 .... .... = Nonce: Not set
        .... 0... .... = Congestion Window Reduced (CWR): Not set
        .... .0.. .... = ECN-Echo: Not set
        .... ..0. .... = Urgent: Not set
        .... ...0 .... = Acknowledgment: Not set
        .... .... 0... = Push: Not set
        .... .... .0.. = Reset: Not set
        .... .... ..1. = Syn: Set
            [Expert Info (Chat/Sequence): Connection establish request (SYN): server port 9876]
                [Connection establish request (SYN): server port 9876]
                [Severity level: Chat]
                [Group: Sequence]
        .... .... ...0 = Fin: Not set
        [TCP Flags: ··········S·]
    Window size value: 65535
    [Calculated window size: 65535]
    Checksum: 0xfe34 [unverified]
    [Checksum Status: Unverified]
    Urgent pointer: 0
    Options: (24 bytes), Maximum segment size, No-Operation (NOP), Window scale, No-Operation (NOP), No-Operation (NOP), Timestamps, SACK permitted, End of Option List (EOL)
        TCP Option - Maximum segment size: 16344 bytes
            Kind: Maximum Segment Size (2)
            Length: 4
            MSS Value: 16344
        TCP Option - No-Operation (NOP)
            Kind: No-Operation (1)
        TCP Option - Window scale: 6 (multiply by 64)
            Kind: Window Scale (3)
            Length: 3
            Shift count: 6
            [Multiplier: 64]
        TCP Option - No-Operation (NOP)
            Kind: No-Operation (1)
        TCP Option - No-Operation (NOP)
            Kind: No-Operation (1)
        TCP Option - Timestamps: TSval 901898369, TSecr 0
            Kind: Time Stamp Option (8)
            Length: 10
            Timestamp value: 901898369
            Timestamp echo reply: 0
        TCP Option - SACK permitted
            Kind: SACK Permitted (4)
            Length: 2
        TCP Option - End of Option List (EOL)
            Kind: End of Option List (0)
```

分析TCP报文内容：

- Source Port 起始端口，当前是51576表示是客户端端口
- Destination Port 目的端口，9876表示是服务器端口
- Sequence number 顺序号码1536975978
- Acknowledgment number 确认号码0
- Flags 各种标志位，其中`SYN`是1

到此，我们就知道TCP三次握手的第一次握手内容：
> 客户端对服务器发起连接，发送 `SYN=1` ，以及 `Seq_Num=X` 

#### 第二次握手

同样选择**序号2**，然后查看TCP报文内容：
- Source Port 起始端口，9876表示从服务器出发
- Destination Port 目的端口，51576
- Sequence number: 3764405294
- Acknowledgment number: 1536975979
- Flags `SYN`和`ACK`是1

> 服务器发送数据给客户端，发送 `SYN=1`，`ACK=1` ，并且收到从客户端传来的 `Seq_Num=X` ，发送 `Ack_Num=X+1` 用于确认，并且发送自身的 `Seq_Num=Y` 

#### 第三次握手

- Source Port 51576
- Destination Port 9876
- Sequence number: 1536975979
- Acknowledgment number: 3764405295
- Flags `ACK`是1

> 客户端发送数据到服务器，发送 `ACK=1`，收到从服务器传来的 `Seq_Num=Y`，发送 `Ack_Num=Y+1` 确认，发送自身的当前顺序号码，第一次握手顺序号码是 `X`，所以这一次的顺序号码 `Seq Num=X+1`

序号为4的内容为`TCP Window Update`，这个用于窗口更新，这里不做赘述。
序号为5的内容如图：
![](https://sonder.vitah.me/ryze/0c40149bf4f990f6fea3066ebdc1f52e.webp)

可以看到，序号5中服务器对客户端发送了数据，数据内容是当时的服务器时间，这里开始就是TCP互相传输内容的部分。

可以用一张图来表现TCP三次握手的过程：
![](https://sonder.vitah.me/ryze/cb88a4b4fca34a33609a25707e1a3370.webp)

### TCP为什么是三次握手

我们已经知道建立连接需要三次握手，那么为什么是三次握手而不是两次？

为了实现可靠的传输，TCP协议的通信双方，都必需维护一个序列号，以标识发送出去的数据包中，哪些是已经被对方收到的。三次握手的过程即时通信双方相互告诉序列号起始值，并确认对方已经收到了序列号起始值的必经步骤。

如果只是两次握手，至多只有连接发起方的起始序列号能被确认，另一方的序列号则得不到确认。譬如发起请求遇到类似这样的情况：客户端发出去的第一个连接请求由于某些原因在网络节点中滞留了导致延迟，直到连接释放的某个时间点才到达服务端，这是一个早已失效的报文，但是此时服务端仍然认为这是客户端的建立连接请求第一次握手，于是服务端回应了客户端，第二次握手。

所以，为了保证服务端能接受到客户端的信息并能作出正确的应答而进行第一次和第二次握手；为了保证客户端能够收到服务端的信息并能作出正确的应答而进行第二次和第三次握手。

三次握手成功才表明双方已经建立的可靠的传输。

### 参考

- [https://www.cnblogs.com/cy568searchx/p/3711670.html](https://www.cnblogs.com/cy568searchx/p/3711670.html)