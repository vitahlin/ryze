---
title: socket基本介绍
slug: basic-introduction-to-socket-in-network-programming
categories:
  - System Programing
tags:
  - 网络
featuredImage: https://sonder.vitah.me/ryze/9641cd3920b7e80dc196f5a99f63b35f.webp
desc: 在这篇博客中，我们将对套接字（Socket）进行基本介绍。套接字是网络编程中的核心概念，它为不同计算机之间的数据交换提供了一种机制。我们将探讨套接字的定义、作用以及常见的套接字类型（如TCP和UDP套接字），并简要介绍如何使用套接字进行网络通信。通过本篇博客，您将对如何使用套接字进行数据传输、如何创建、绑定和监听套接字等基本操作有一个清晰的理解，并为进一步的网络编程打下基础。
draft: false
---

## 问题

1. 网络中如何唯一标识一个进程
2. 什么是socket
3. socket相关的基本函数
4. socket基本用法

### 网络中如何唯一标识一个进程

在本地我们可以通过进程 PID 来唯一标识一个进程，在网络中，TCP/IP 协议族则用 IP 地址来唯一标识网络中的主机，而传输层的“协议+端口”标识主机中的应用程序（进程）。这样利用三元组（IP 地址，协议，端口）就可以标识网络的进程。

## 什么是socket

网络中的进程是通过 socket 来通信的，那么什么是 socket？

socket 是⽤户进程和内核⽹络协议栈之间的编程接⼝，其实就是操作系统提供给操作「⽹络协议栈」的接⼝，你能通过 socket 的接⼝，来控制协议栈⼯作，从⽽实现网络通信，达到跨主机通信。

假如要编程网络程序，进行服务器端和客户端的通信（数据交换）。不使用 socket 的话，需要做下面的一堆事情：
1. 管理缓存区来接收和发送数据；
2. 告诉操作系统自己要监听某个端口的数据，还有自己处理这些系统调用来读取数据；
3. 当没有连接的时候或者另外一端要还没有发送数据时候，要处理 IO 阻塞，把自己挂起；
4. 封装和解析 TCP/IP 协议的数据，维护数据发送的顺序；
5. 等等。

做了一大堆东西，发现最重要的还没有做：发送/接收数据。如果有一个程序能够自动帮我们把上面的东西都做掉，这样我们就可以只关心数据的读写，编程就简单的多了。
这样一个程序就是 socket，它现在已经是操作系统的一部分，在 Linux 中是标准的系统调用，只要调用它提供的一组接口（下面会详解常用函数的使用），就能轻松地建立连接， 读写数据，关闭连接，让网络操作就像文件操作一样简单。
这里也体现了Unix的哲学*一切皆文件 。* 

socket 基本流程图：
![socket基本流程图](https://sonder.vitah.me/ryze/a203a10884aef8b50ad1c3c7cf007e87.webp)

## 相关函数介绍

### socket函数

`socket()` 函数用于创建一个网络套接字(socket)。套接字是网络通信的端点，它允许应用程序通过网络进行数据交换。`socket()` 函数是用于网络编程的基础，通常用于建立客户端和服务器之间的连接。

函数原型：
```c
#include <sys/types.h>          
#include <sys/socket.h>
int socket(int domain, int type, int protocol);
```

当我们创建一个 socket 时，返回的 socket 描述字它存在于协议族 `(address family，AF_XXX)` 空间中，但没有一个具体的地址。如果想要给它赋值一个地址，就必须调用 `bind()` 函数，否则就当调用 `connect()` 、`listen()` 时系统会自动随机分配一个端口。

- `domain`：地址族，`domain` 用来设置网络通信的域名，也就是 IP 地址类型，常用的有 `AF_INET` 和 `AF_INET6` ，`AF_INET` 表示 IPv4 地址，`AF_INET6` 表示 IPv6 地址。
-   `type`：数据传输方式/套接字类型，常用的有 `SOCK_STREAM(流格式套接字/面向连接的套接字)` 和 `SOCK_DGRAM(数据报套接字/无连接的套接字)` 。
-   `protocol`：表示传输协议，常用的有 `IPPROTO_TCP` 和 `IPPTOTO_UDP` ，分别表示 TCP 传输协议和 UDP 传输协议。可以将 *protocol 的值设为 0，系统会自动推演出应该使用什么协议*。

函数 `socket()` 并不总是执行成功，有可能会出现错误，错误的产生有多种原因，可以通过 `errno` 获得。

使用示例：
```c
// 创建tcp套接字
int tcp_socket = socket(AF_INET, SOCK_STREAM, 0);  
// 创建unp套接字
int udp_socket = socket(AF_INET, SOCK_DGRAM, 0);
```

### connect函数

TCP客户端用 `connect()` 函数来建立与 TCP 服务器的连接。

```c
#include<sys/socket.h>
int connect (int sockfd, const struct sockaddr * serv_addr, socklen_t addrlen);
```

-   `sockfd`: 由 `socket()` 函数返回的套接字描述符
-   `serv_addr`: 套接字地址结构的指针。套接字地址结构必须含服务器的 IP 地址和端口号。
-   `addrlen`: 套接字地址结构的大小

客户端在调用 `connect()` 前不必非得调用 `bind()` ，因为如果需要的话，内核会确定源 IP 地址，并选择一个临时端口作为源端口。

如果是 TCP 套接字，调用 `connect()` 将激发 TCP 的三路握手过程，而且仅在连接建立成功或出错时才返回，其中出错返回可能由以下几种情况：
1.  客户端没有收到 `SYN` 的响应，返回 `ETIMEDOUT` 错误，表示连接不可达。例如指定一个并不存在的IP地址。
2.  对客户的响应是 `RST(表示复位)` ，则表示服务器在指定的端口上没有进程等待连接（例如服务器进程没有在运行）。客户端一收到 `RST` 返回 `ECONNREFUSED` 错误。
3.  客户端发出的 `SYN` 在某个路由器上面引发了一个“**目的地不可达**” ICMP 错误。例如，指定一个因特网中不可到达的IP地址。

### bind函数

`bind()` 用于将一个套接字（socket）与一个本地地址（包括 IP 地址和端口号）绑定。
在网络编程中，服务器端需要将其套接字绑定到一个特定的 IP 地址和端口上，以便客户端能够通过该地址和端口连接到服务器。

```c
#include<sys/socket.h>
int bind ( int sockfd, struct sockaddr * addr, socklen_t addrlen )
```

- `sockfd`: 指定地址与哪个套接字绑定，这是一个由之前的 `socket()` 调用返回的套接字。调用 `bind()` 之后，该套接字与一个相应的地址关联，发送到这个地址的数据可以通过这个套接字来读取与使用；
- `addr`: 已经经过填写的有效的地址结构；
- `addrlen`: 地址长度。

返回值：成功返回0，失败返回-1，并设置 errno 来表示错误原因。

`bind()` 函数并不是总需要调用，只要用户进程想与一个具体的地址或端口相关联的时候才需要调用这个函数。
如果用户进程没有调用，程序可以依赖内核的自动选址机制来完成自动地址选择。**一般情况下，对服务器进程需要调用 `bind()` ，对客户端进程则不需要调用 `bind()` 。**

示例：
```c
```c
if ((listen_fd = socket(AF_INET, SOCK_STREAM, 0)) < 0)
{
    printf("socket error");
    exit(-1);
}

bzero(&serv_addr, sizeof(serv_addr));

serv_addr.sin_family = AF_INET;
serv_addr.sin_port = htons(9876);
serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);

if (bind(listen_fd, (const struct sockaddr *) &serv_addr, sizeof(serv_addr)) < 0)
{
    printf("bind error");
    exit(-1);
}
```

对于 IPv4 来说，通配地址通常由 `INADDR_ANY` 来指定，其值一般为 `0.0.0.0` ，表示任意地址。

### listen函数

`listen()` 函数仅有 TCP 服务端调用，用于将一个已绑定的套接字转换为监听状态。它是服务器端用于等待客户端连接请求的关键步骤。通过调用 `listen()` 服务器告诉操作系统，它准备接受来自客户端的连接。

```c
#include<sys/socket.h>
int listen ( int sockfd,  int backlog )
```

成功时，返回 0；失败时，返回 -1，并设置 errno 来表示错误原因。

它做两件事情：
1.  当 `socket()` 创建一个套接字时，它被假设成一个主动套接字，就是说它是一个将调用 `connect()` 发起连接的客户端套接字。`listen()` 则把一个未连接套接字转换成一个被动套接字，指示内核应接受该套接字的连接请求。`调用 listen() 将把套接字从 CLOSED 状态转换到 LISTEN 状态；`
2.  第二个参数规定了内核应该为相应套接字排队的最大连接个数。

如何理解参数 backlog？
> 内核为任何一个给定的监听套接字维护两个队列：
> 1. 未完成连接队列。服务区正在等待完成相应的TCP三次握手连接。
> 2. 已完成连接队列。每个已经完成的TCP三次握手过程的客户对应其中一项。
上述两个队列之和不超过 backlog。

### accept函数

在调用 `accept()` 函数之前，服务器端必须先调用 `socket()` 创建套接字，并通过 `bind()` 将套接字绑定到指定的地址和端口，并通过 `listen()` 启动监听。

`accept()` 由服务器调用，它从连接请求队列中获取一个待处理的连接，并为该连接创建一个新的套接字，之后服务器可以使用这个新的套接字与客户端进行通信。

默认情况下，**套接字（socket）是阻塞模式**。如果这时候请求队列为空，`accept()` 函数会发生阻塞，直到有客户端请求连接并进入队列。如果没有连接请求到达，`accept()` 就会一直等待，直到有新的连接请求被排入队列。

```c
#include <sys/socket.h>
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

- `sockfd`：是由 `socket()` 返回的套接字描述符；
- `addr`：指向 `struct sockaddr` 结构体的指针，用来保存连接的客户端的地址信息（例如客户端的 IP 地址和端口号），若不需要获取客户端的地址信息，可以将其设置为 NULL。
- `addrlen`：一个指向 `socklen_t` 类型变量的指针，用来传入 addr 缓冲区的大小，并返回实际的地址长度。可以设置为 NULL，如果不关心地址信息。

返回值：
- 如果成功，返回一个新的套接字文件描述符，这个套接字用于与客户端进行通信。在 `accept()` 函数中，我们称第一个参数 sockfd 为**监听套接字描述符**，称它的返回值为**已连接套接字描述符**。一个服务器通常仅创建一个监听套接字，它在该服务器的生命周期内一直存在。内核为每个服务器进程接受的客户连接创建一个已连接套接字，当服务器完成对某个给定客户的服务时，已连接套接字就被关闭。
- 失败返回 -1，并设置 errno 来表示错误。

### close函数

```c
int close(int sockfd); 
```

功能：
- 释放资源：会释放套接字占用的内存、连接状态等；
- 关闭套接字：对于网络套接字，调用 close() 后，套接字的连接会被关闭，不能再进行读写操作。该套接字描述符不能再由调用进程使用，也就是它不能再作为 `read` 或 `write` 的第一个参数；
- 通知远程端：在 TCP 协议下，调用 `close()` 会发送一个 `TCP FIN(结束)` 包给远程端，表示本端已经完成数据发送，准备断开连接。

在多进程并发服务器中，父子进程共享套接字，套接字描述符引用计数记录着共享着的进程个数，当父进程或某一子进程 `close` 套接字时，描述符引用计数会相应的减1，**当引用计数仍然大于0时，这个 close 调用不会引发 TCP 四次挥手过程。

## socket代码示例

下面是一段示例，服务端把当前时间返回给客户端。代码中的 `wrapXXX()` 函数都是**对原函数XXX的封装**，比如 `wrapSocket()` 和 `wrapBind()` 具体实现如下，封装了调用 `socket()` 和 `bind()` 时错误的处理：
```c
int wrapSocket(int domain, int type, int protocol) {  
    int n;  
    if ((n = socket(domain, type, protocol)) < 0) {  
        perror("socket error ");  
        exit(-1);  
    }  
  
    return n;  
}

void wrapBind(int sock_fd, const struct sockaddr *address, socklen_t sock_len) {  
    if (bind(sock_fd, address, sock_len) < 0) {  
        perror("bind error ");  
        exit(-1);  
    }  
}
```

服务端代码：
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
        // 写入内容后，关闭
        wrapClose(conn_fd);  
    }  
}
```

客户端代码：
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

运行结果：
```shell
> ./echo_time_tcp_srv_ch1_9
time server running...
New client connect IP=0.0.0.0, port=0, conn_id=4
# 关闭客户端后，服务器程序还是在运行...
```

```shell
> ./get_time_tcp_cli_ch1_5
Input IP address: 127.0.0.1
Input port: 9876
Wed Mar 26 21:05:29 2025
```
