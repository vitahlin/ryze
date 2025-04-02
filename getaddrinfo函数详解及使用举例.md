---
title: getaddrinfo函数详解及使用举例
slug: getaddrinfo-introduction-and-usage-examples
categories:
  - Lang
tags:
  - C
  - 网络
featuredImage: https://sonder.vitah.me/ryze/01e82c8384b61e4ea88e9ec75ac23ad2.webp
desc: getaddrinfo 是 C 语言中用于解析主机名和服务信息的重要函数，它提供了一种统一的方式来获取 IPv4 和 IPv6 地址信息，广泛用于网络编程。本文将详细介绍 getaddrinfo 的参数、返回值以及常见的错误处理，并提供完整的示例代码，帮助读者理解如何正确使用 getaddrinfo 进行地址解析和套接字创建。
draft: false
---

## 函数介绍

IPv 4 中使用 `gethostbyname()` 函数完成主机名到地址解析，这个函数仅仅支持 IPv 4，且不允许调用者指定所需地址类型的任何信息，返回的结构只包含了用于存储 IPv 4 地址的空间。IPv 6 中引入了 `getaddrinfo()` 的新 API，它是协议无关的，既可用于 IPv 4 也可用于 IPv 6。
`getaddrinfo` 函数能够处理名字到地址以及服务到端口这两种转换，返回的是一个 `addrinfo` 的结构（列表）指针而不是一个地址清单。
这些 `addrinfo` 结构随后可由 `socket()` 函数直接使用。如此以来，` getaddrinfo ` 函数把协议相关性安全隐藏在这个库函数内部。应用程序只要处理由 ` getaddrinfo ` 函数填写的套接口地址结构。该函数在 POSIX 规范中定义了。

函数原型：
```c
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>
int getaddrinfo( const char *hostname, const char *service, const struct addrinfo *hints, struct addrinfo **result );
```

参数说明：
- `hostname`: 一个主机名或者地址串 (IPv 4 的点分十进制串或者 IPv 6 的 16 进制串)。
    - 主机名例如 "www.baidu.com"
    - IP 地址（例如 “192.168.1.1” 或 “2001: db 8:: 1”）
    - 如果为 NULL，则 `getaddrinfo` 会返回本地地址（使用 `INADDR_ANY` 或 `IN6ADDR_ANY_INIT`）。
- `service`：服务名可以是十进制的端口号，也可以是已定义的服务名称，如 ftp、http 等。
    - 服务名称（例如 “http” 或 “ftp”）；
    - 端口号（如 “80” 或 “443”）。
    - 如果为 NULL，则表示不需要解析端口。
- `hints`：一个指向 `struct addrinfo` 的指针，用于提供给 `getaddrinfo` 解析的提示（例如是否使用 IPv 4 或 IPv 6，是否使用流式套接字等）。可以根据需要进行设置，否则设置为全零结构，表示不限制。
- `result`：一个指向 `struct addrinfo *` 的指针，用于存储解析结果。`getaddrinfo()` 会将返回的地址信息**以链表的形式存储**在这个指针指向的区域，每个链表节点包含一个网络地址（例如 `sockaddr_in` 或 `sockaddr_in6`）。

返回值
- 成功时，返回 0，并且 `result` 会指向链表的第一个元素（存储了地址信息）；
- 失败时，返回一个非零值，具体的错误码可以通过 `gai_strerror()` 获取错误描述。常见错误代码：
    - `EAI_NONAME`：没有找到与给定名称匹配的地址；
    - `EAI_AGAIN`：临时的 DNS 错误或网络不可达；
    - `EAI_FAIL`：请求的 DNS 查询失败。

## 详细说明 

`getaddrinfo()`  返回的地址信息会保存在一个 `struct addrinfo`  结构体中，该结构体通常具有以下字段：
```c
struct addrinfo {
    int ai_flags;          // 标志位，例如 AI_PASSIVE, AI_CANONNAME 等
    int ai_family;         // 地址族（如 AF_INET 表示 IPv4，AF_INET6 表示 IPv6）
    int ai_socktype;       // 套接字类型（如 SOCK_STREAM 表示 TCP）
    int ai_protocol;       // 协议类型（如 IPPROTO_TCP）
    size_t ai_addrlen;     // 地址长度
    char *ai_canonname;    // 规范化后的主机名
    struct sockaddr *ai_addr;  // 地址信息，通常为 sockaddr_in 或 sockaddr_in6
    char *ai_serv;         // 服务名或端口号（如 "http" 或 "80"）
    struct addrinfo *ai_next;  // 下一个 addrinfo 结构的指针（链表）
};
```

hints 用来指定你希望解析的地址类型，通常是这样配置：
```c
struct addrinfo hints;
memset(&hints, 0, sizeof(hints));
hints.ai_family = AF_UNSPEC; // 不限制地址族（可以是 IPv4 或 IPv6）
hints.ai_socktype = SOCK_STREAM; // TCP 套接字类型
hints.ai_flags = AI_PASSIVE; // 对于服务器，绑定到任意可用地址
```

- `ai_family` 可以设置为 `AF_INET(IPv4)` 或 `AF_INET6(IPv6)` ，也可以设置为 `AF_UNSPEC` ，表示不限制地址族。
- `ai_socktype` 通常设置为 `SOCK_STREAM(TCP)` 或 `SOCK_DGRAM(UDP)` 。
- `ai_flags` 用来控制解析的行为，常见的标志有 `AI_PASSIVE`（用于绑定到任意可用接口）和 `AI_CANONNAME` （获取规范化的主机名）。

### AI_PASSIVE 在 getaddrinfo 中的作用

`getaddrinfo()` 是一个用于解析地址的函数，它通常用来为客户端或服务器选择合适的地址来进行连接或绑定。如果你是服务器端程序，通常会希望它能绑定到所有可用的网络接口，处理来自任何网络接口的请求。这时，你可以调用 `getaddrinfo()` 并传递 `AI_PASSIVE` 标志。这样，如果没有指定特定的地址（如在 bindaddr 参数为空或 NULL 的情况下），函数会返回适合绑定到所有可用接口的地址。

**AI_PASSIVE 的作用：**
- **没有指定地址时**：如果 `bindaddr` 参数为空或 NULL，`AI_PASSIVE` 会导致 `getaddrinfo()` 返回一个适用于绑定到所有可用网络接口的地址。对 IPv 4 地址来说，就是 `INADDR_ANY`（表示可以绑定到所有可用的网络接口，通常是 0.0.0.0）。对 IPv 6 地址来说，就是 `IN6ADDR_ANY_INIT`（通常表示 ::）。
- **已经指定了地址时**：如果 `bindaddr` 已经指定了某个特定的地址，`AI_PASSIVE` 标志不会有任何效果。它只是作为一个提示，表示如果没有明确指定地址，应该选择一个通用的地址。

示例代码：
```c
struct addrinfo hints, *servinfo;
memset(&hints, 0, sizeof(hints));
hints.ai_family = AF_INET;  // IPv4
hints.ai_socktype = SOCK_STREAM; // TCP
hints.ai_flags = AI_PASSIVE;  // 如果没有指定地址，返回适用于所有接口的地址

// 获取服务器端地址信息，监听所有接口
if (getaddrinfo(NULL, "8080", &hints, &servinfo) != 0) {
    perror("getaddrinfo");
    return 1;
}

// 创建并绑定套接字
int sockfd = socket(servinfo->ai_family, servinfo->ai_socktype, servinfo->ai_protocol);
bind(sockfd, servinfo->ai_addr, servinfo->ai_addrlen);
```

Redis 中对 getaddrinfo 的使用：
```c
static int _anetTcpServer(char *err, int port, char *bindaddr, int af, int backlog)  
{  
    int s = -1, rv;  
    char _port[6];  /* strlen("65535") */  
    struct addrinfo hints, *servinfo, *p;  
  
    snprintf(_port,6,"%d",port);  
    memset(&hints,0,sizeof(hints));  
    hints.ai_family = af;  
    hints.ai_socktype = SOCK_STREAM;  
    hints.ai_flags = AI_PASSIVE;    /* No effect if bindaddr != NULL */  
    if (bindaddr && !strcmp("*", bindaddr))  
        bindaddr = NULL;  
    if (af == AF_INET6 && bindaddr && !strcmp("::*", bindaddr))  
        bindaddr = NULL;  
  
    if ((rv = getaddrinfo(bindaddr,_port,&hints,&servinfo)) != 0) {  
        anetSetError(err, "%s", gai_strerror(rv));  
        return ANET_ERR;  
    }  
    for (p = servinfo; p != NULL; p = p->ai_next) {  
        if ((s = socket(p->ai_family,p->ai_socktype,p->ai_protocol)) == -1)  
            continue;  
  
        if (af == AF_INET6 && anetV6Only(err,s) == ANET_ERR) goto error;  
        if (anetSetReuseAddr(err,s) == ANET_ERR) goto error;  
        if (anetListen(err,s,p->ai_addr,p->ai_addrlen,backlog,0) == ANET_ERR) s = ANET_ERR;  
        goto end;  
    }  
    if (p == NULL) {  
        anetSetError(err, "unable to bind socket, errno: %d", errno);  
        goto error;  
    }  
  
error:  
    if (s != -1) close(s);  
    s = ANET_ERR;  
end:  
    freeaddrinfo(servinfo);  
    return s;  
}
```
`bindaddr` 是 Redis 配置的绑定地址。
- `*` 代表所有 IPv4 地址，转换为 NULL 以监听 `0.0.0.0` 。 
- `::*` 代表所有 IPv6 地址，转换为 NULL 以监听 `::` （即所有 IPv6 地址）

后续遍历 `getaddrinfo` 返回的 `addrinfo` 链表，尝试创建 `socket`，如果创建失败，继续尝试下一个地址。

## 示例代码

### 1. 根据主机名获取地址信息 

```c
#include <stdio.h>  
#include <netdb.h>  
#include <string.h>  
#include <arpa/inet.h>  
  
int main() {  
    struct addrinfo hints, *result, *p;  
    int status;  
  
    // 设置hint结构体数据全为0，不然可能出现随机值  
    memset(&hints, 0, sizeof(hints));  
  
    // 不限制IPv4 IPv6  
    hints.ai_family = AF_UNSPEC;  
    // TCP套接字类型  
    hints.ai_socktype = SOCK_STREAM;  
  
    if ((status = getaddrinfo("www.baidu.com", "http", &hints, &result)) != 0) {  
        fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(status));  
        return 1;  
    }  
  
    // 遍历所有返回的地址信息  
    for (p = result; p != NULL; p = p->ai_next) {  
        void *addr;  
        char ip_str[INET6_ADDRSTRLEN];  
  
        // 获取对应的IPv4或IPv6地址  
        if (p->ai_family == AF_INET) {  
            struct sockaddr_in *ipv4 = (struct sockaddr_in *) p->ai_addr;  
            addr = &(ipv4->sin_addr);  
        } else {  
            struct sockaddr_in6 *ipv6 = (struct sockaddr_in6 *) p->ai_addr;  
            addr = &(ipv6->sin6_addr);  
        }  
  
        inet_ntop(p->ai_family, addr, ip_str, sizeof(ip_str));  
        printf("IP address:%s\n", ip_str);  
    }  
  
    freeaddrinfo(result); // 释放内存  
  
    return 0;  
}
```

运行结果：
```shell
IP address:157.148.69.186
IP address:157.148.69.151
```

### 2. 服务器程序获取可绑定端口和地址

```c
#include <stdio.h>
#include <netdb.h>
#include <string.h>
#include <arpa/inet.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdarg.h>
#include <string.h>
#include <fcntl.h>
#include <time.h>
#include <ctype.h>
#include <unistd.h>
#include <errno.h>
#include <sys/wait.h>
#include <signal.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <sys/stat.h>
#include <netdb.h>
#include <sys/socket.h>


void getCanBindAddressInTcpServer() {
    struct addrinfo hints, *res;
    int sockfd;

    // 1. 清空 hints 并设置参数
    memset(&hints, 0, sizeof hints);
    hints.ai_family = AF_UNSPEC;    // 支持 IPv4 或 IPv6
    hints.ai_socktype = SOCK_STREAM; // 选择 TCP 连接
    hints.ai_flags = AI_PASSIVE;     // 监听所有可用 IP (0.0.0.0)

    // 2. 获取可用的地址
    int status = getaddrinfo(NULL, "8080", &hints, &res);
    if (status != 0) {
        fprintf(stderr, "getaddrinfo error: %s\n", gai_strerror(status));
        return;
    }

    // 3. 创建 socket
    sockfd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);
    if (sockfd == -1) {
        perror("socket error");
        return;
    }

    // 4. 绑定端口
    if (bind(sockfd, res->ai_addr, res->ai_addrlen) == -1) {
        perror("bind error");
        close(sockfd);
        return;
    }

    printf("Server is running on port 8080...\n");

    // 5. 释放资源
    freeaddrinfo(res);
}

int main() {
    printf("\ngetCanBindAddressInTcpServer:\n");
    getCanBindAddressInTcpServer();
    return 0;
}

```

运行结果：
```shell
getCanBindAddressInTcpServer:
Server is running on port 8080...
```