---
title: getaddrinfo函数
slug: 
categories: 
tags: 
featuredImage: 
desc: 
draft: true
---

## 函数介绍

IPv4中使用 `gethostbyname()` 函数完成主机名到地址解析，这个函数仅仅支持 IPv4，且不允许调用者指定所需地址类型的任何信息，返回的结构只包含了用于存储IPv4地址的空间。IPv6中引入了 `getaddrinfo()` 的新API，它是协议无关的，既可用于IPv4也可用于IPv6。
`getaddrinfo` 函数能够处理名字到地址以及服务到端口这两种转换，返回的是一个 `addrinfo` 的结构（列表）指针而不是一个地址清单。
这些 `addrinfo` 结构随后可由套接口函数直接使用。如此以来，`getaddrinfo` 函数把协议相关性安全隐藏在这个库函数内部。应用程序只要处理由 `getaddrinfo` 函数填写的套接口地址结构。该函数在 POSIX规范中定义了。

`getaddrinfo` 是一个用于解析主机名、服务名以及相关网络信息的函数，它被广泛用于网络编程中，尤其是当需要处理不同协议（如 IPv4 和 IPv6）时。`getaddrinfo()` 的一个重要特点是它能够根据系统的配置自动选择适当的地址族（例如 IPv4 或 IPv6），并为后续的套接字操作提供必要的地址信息。

函数原型：
```c
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>
int getaddrinfo( const char *hostname, const char *service, const struct addrinfo *hints, struct addrinfo **result );
```

参数说明：
- `hostname`: 一个主机名或者地址串(IPv4 的点分十进制串或者 IPv6 的 16 进制串)。
    - 主机名例如 “www.example.com“；
    - IP 地址（例如 “192.168.1.1” 或 “2001:db8:: 1”）
    - 如果为 NULL，则 `getaddrinfo` 会返回本地地址（使用 `INADDR_ANY` 或 `IN6ADDR_ANY_INIT`）。
- `service`：服务名可以是十进制的端口号，也可以是已定义的服务名称，如 ftp、http 等。
    - 服务名称（例如 “http” 或 “ftp”）；
    - 端口号（如 “80” 或 “443”）。
    - 如果为 NULL，则表示不需要解析端口。
- `hints`：一个指向 `struct addrinfo` 的指针，用于提供给 `getaddrinfo` 解析的提示（例如是否使用 IPv4 或 IPv6，是否使用流式套接字等）。可以根据需要进行设置，否则设置为全零结构，表示不限制。
- `result`：一个指向 `struct addrinfo *` 的指针，用于存储解析结果。`getaddrinfo()` 会将返回的地址信息**以链表的形式存储**在这个指针指向的区域。每个链表节点包含一个网络地址（例如 `sockaddr_in` 或 `sockaddr_in6`）。

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

### AI_PASSIVE 在 getaddrinfo () 中的作用

`AI_PASSIVE` 是 `getaddrinfo()`  函数中的一个标志（flag），主要用于服务器端的地址解析。
当使用 `getaddrinfo()` 获取地址信息时，如果你设置了 `AI_PASSIVE` 标志，它会在没有明确指定地址（即 bindaddr 参数为空或 NULL）时，自动返回一个适合绑定到所有可用网络接口的通用地址。对于 IPv4，它会返回 INADDR_ANY（即 0.0.0.0）；对于 IPv6，它会返回 IN6ADDR_ANY_INIT（即 ::）。

- getaddrinfo() 是一个用于解析地址的函数，它通常用来为客户端或服务器选择合适的地址来进行连接或绑定。
- 如果你是服务器端程序，通常会希望它能绑定到所有可用的网络接口，处理来自任何网络接口的请求。这时，你可以调用 `getaddrinfo()` 并传递 `AI_PASSIVE` 标志。这样，如果没有指定特定的地址（如在 bindaddr 参数为空或 NULL 的情况下），函数会返回适合绑定到所有可用接口的地址。

**AI_PASSIVE 的作用：**
- **没有指定地址时**：如果 `bindaddr` 参数为空或 NULL，`AI_PASSIVE` 会导致 `getaddrinfo()` 返回一个适用于绑定到所有可用网络接口的地址。对 IPv4 地址来说，就是 `INADDR_ANY`（表示可以绑定到所有可用的网络接口，通常是 0.0.0.0）。对 IPv6 地址来说，就是 `IN6ADDR_ANY_INIT`（通常表示 ::）。
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

示例代码

根据主机名获取地址信息：
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

