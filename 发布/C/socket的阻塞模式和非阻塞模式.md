---
title: socket的阻塞模式和非阻塞模式
slug: socket-blocking-vs-nonblocking
categories:
  - System Programing
tags:
  - 网络
featuredImage: https://sonder.vitah.me/ryze/87315cc5d4dc975ca631b854ed723d84.webp
desc: 本文深入对比了 socket 的阻塞模式与非阻塞模式的行为差异，涵盖 connect、accept、send、recv 等常用函数在两种模式下的表现，以及各自的优缺点、适用场景和错误处理方式，帮助你在开发高性能网络程序时做出正确选择。
draft: false
---

## 阻塞模式和非阻塞模式介绍

什么是阻塞 socket，什么是非阻塞 socket。对于这个问题，我们要先弄清什么是阻塞/非阻塞。阻塞与非阻塞是对一个文件描述符指定的文件或设备的两种工作方式。
- 阻塞模式，就当某个函数“执行成功的条件”当前不能满足时，该函数会阻塞当前执行线程，程序执行流在超时时间到达或“执行成功的条件”满足后恢复继续执行。比如对该文件描述符进行读写时，如果当时没有东西可读或者暂时不可写，程序就进入等待状态，直到有东西可读或者可写为止。
- 非阻塞模式恰恰相反，即使某个函数的“执行成功的条件”不当前不能满足，该函数也不会阻塞当前执行线程，而是立即返回，继续运行执行程序流。当没有东西可读或者不可写时，读写函数就马上返回，而不会等待。

无论是 Windows 还是 Linux 平台，**默认创建的 socket 都是阻塞模式的**。不过我们可以通过函数 `fcntl()` 给 socket 增加 **O_NONBLOCK** 标志来设置成非阻塞模式。

`fcntl` 是一个在 Unix/Linux 系统中用于操作文件描述符的系统调用函数，它常用于：
1. 设置/获取文件描述符的属性（如阻塞/非阻塞模式）
2. 锁定文件（文件锁）
3. 复制文件描述符
4. 设置文件描述符标志（如 FD_CLOEXEC）

函数原型：
```c
#include <unistd.h> 
#include <fcntl.h> 
int fcntl(int fd, int cmd, ...);
```
- fd：要操作的文件描述符
- cmd：操作命令
- arg：可选参数，根据 cmd 不同会有不同的类型

常见的 CMD 命令：

| 命令                         | 作用                      |
| -------------------------- | ----------------------- |
| F_GETFL                    | 获取文件描述符状态标志（如是否是非阻塞）    |
| F_SETFL                    | 设置文件描述符状态标志             |
| F_GETFD                    | 获取文件描述符标志（如 FD_CLOEXEC） |
| F_SETFD                    | 设置文件描述符标志               |
| F_DUPFD                    | 复制文件描述符，返回新描述符          |
| F_SETLK, F_GETLK, F_SETLKW | 设置或获取文件锁                |

示例代码，socket 设置为非阻塞模式：
```c
int oldSocketFlag = fcntl(sockfd, F_GETFL, 0);
int newSocketFlag = oldSocketFlag | O_NONBLOCK;
fcntl(sockfd, F_SETFL,  newSocketFlag);
```

示例代码，加写锁：
```c
struct flock fl;
fl.l_type = F_WRLCK;
fl.l_whence = SEEK_SET;
fl.l_start = 0;
fl.l_len = 0; // 锁整个文件

if (fcntl(fd, F_SETLK, &fl) == -1) {
    perror("Lock failed");
}
```

## socket阻塞和非阻塞有哪些影响

阻塞和非阻塞模式下，我们常讨论的具有不同行为表现的 socket 函数一般有如下几个：
- connect
- accept
- send (Linux 平台上对 socket 进行操作时也包括 **write** 函数，下文中对 send 函数的讨论也适用于 **write** 函数)
- recv (Linux 平台上对 socket 进行操作时也包括 **read** 函数，下文中对 recv 函数的讨论也适用于 **read** 函数)

### 建立连接 connect

阻塞情况下，`connect` 首先发送 `SYN` 请求到服务器，当客户端收到服务器返回的 `SYN` 的确认时，则 `connect` 返回，否则的话一直阻塞。
```c
int sockfd = socket(AF_INET, SOCK_STREAM, 0);
connect(sockfd, (struct sockaddr *)&addr, sizeof(addr)); // 阻塞调用
```

上述代码会**阻塞等待连接完成**。如果服务器很慢，或者网络有问题，**程序就会卡在这里直到连接成功或失败（超时）**，适合简单、同步式逻辑。

非阻塞模式下，`connect()` 将启用 TCP 协议的三次握手，但是 `connect()` 并不等待连接建立好才返回，而是立即返回：
```c
int flags = fcntl(sockfd, F_GETFL, 0);
fcntl(sockfd, F_SETFL, flags | O_NONBLOCK);

int ret = connect(sockfd, (struct sockaddr *)&addr, sizeof(addr));
if (ret < 0 && errno == EINPROGRESS) {
    // 正在连接中，不是错误！
    // 后续你可以用 select()/poll()/epoll() 等等待写事件
}
```

上述代码中 `connect()` 会**立即返回**，不会阻塞程序。
- 如果连接正在建立，会返回 `-1`，并设置 `errno` 为 `EINPROGRESS` 。
- 后续需要使用 `select()`、`poll()` 或 `epoll()` 等 **I/O 多路复用机制**来等待连接完成。
- 连接成功后，socket 会变得可写；可以用 `getsockopt()` 判断是否成功。

用 `select()` 和 ` getsockopt() ` 来判断 socket 是否可写：
```c
fd_set writefds;
FD_ZERO(&writefds);
FD_SET(sockfd, &writefds);
struct timeval timeout = {5, 0}; // 5 秒超时


int ready = select(sockfd + 1, NULL, &writefds, NULL, &timeout);
if (ready > 0 && FD_ISSET(sockfd, &writefds)) {
    int err;
    socklen_t len = sizeof(err);
    getsockopt(sockfd, SOL_SOCKET, SO_ERROR, &err, &len);
    if (err == 0) {
        // 连接成功
    } else {
        // 连接失败
    }
}
```

### 函数 accept

```c
int sockfd = socket(AF_INET, SOCK_STREAM, 0);
connect(sockfd, (struct sockaddr *)&addr, sizeof(addr)); // 阻塞调用
```

会**阻塞等待连接完成**，如果服务器很慢，或者网络有问题，**程序就会卡在这里直到连接成功或失败（超时）**。

```c
int flags = fcntl(listenfd, F_GETFL, 0);
fcntl(listenfd, F_SETFL, flags | O_NONBLOCK);

int connfd = accept(listenfd, NULL, NULL);
if (connfd == -1 && errno == EAGAIN) {
    // 当前没有连接，不是错误，只是暂时没有客户端连接
}
```

上述代码不会阻塞主线程，如果没有连接到达，`accept()` 立即返回 -1，`errno` 为 `EAGAIN` 或 `EWOULDBLOCK` ，常与 `select/poll/epoll` 等 I/O 多路复用机制结合使用，只在监听 fd 可读时再调用 `accept()` ，示例代码：
```c
if (FD_ISSET(listenfd, &readfds)) {
    int connfd = accept(listenfd, NULL, NULL);
    if (connfd == -1) {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            // 没有连接，忽略
        } else {
            perror("accept error");
        }
    } else {
        // 有新连接，处理 connfd
    }
}
```

需要注意的是，非阻塞模式下，要做好循环 `accept`（有可能一次触发时有多个连接进来）：
```c
while ((connfd = accept(listenfd, NULL, NULL)) != -1) {
    // 处理 connfd
}
// errno == EAGAIN 时退出
```

### send 函数

```c
ssize_t ret = send(sockfd, buf, len, 0);
```

阻塞模式下，当 socket 的发送缓冲区满时，`send()` 会阻塞，直到有足够空间写入全部数据，或者发送部分数据。如果系统信号打断了 `send()`（比如收到 `SIGINT` ），它会返回 `-1` 并设置 ` errno=EINTR ` 。

非阻塞模式下：
```c
ssize_t ret = send(sockfd, buf, len, 0);
if (ret == -1) {
    if (errno == EAGAIN || errno == EWOULDBLOCK) {
        // 发送缓冲区已满，不能立即发送数据
    } else {
        perror("send error");
    }
}
```

如果发送缓冲区满，**不会阻塞**，而是直接返回 `-1` ，并设置 ` errno ` 为 ` EAGAIN ` 或 ` EWOULDBLOCK ` 。此时需要等待 socket **变成可写**时再重试发送（通过 ` select/poll/epoll ` 等机制）。

这种情况下需要**管理发送缓冲区**，有可能已经写了一部分，还要继续写剩下的内容：
```c
// 伪代码：循环写直到全部发送完
while (send_pos < total_len) {
    ssize_t ret = send(sockfd, buf + send_pos, total_len - send_pos, 0);
    if (ret > 0) {
        send_pos += ret;
    } else if (ret == -1 && (errno == EAGAIN || errno == EWOULDBLOCK)) {
        // 当前不可写，等待事件触发再继续写
        break;
    } else {
        // 错误处理
        break;
    }
}
```

### recv 函数

```c
ssize_t n = recv(sockfd, buf, len, 0);
```

阻塞模式下，如果 socket 接收缓冲区 **没有数据**，`recv()` 会 **阻塞当前线程**，直到：
1. 有数据可读；
2. 对方关闭连接；
3. 调用被信号中断（返回 -1，`errno=EINTR` ）。

返回值：
1. 大于 0，实际读取的字节数
2. 0，对方已关闭连接（FIN）
3. -1，发生错误，如被信号打断（`errno=EINTR` ）

```c
ssize_t n = recv(sockfd, buf, sizeof(buf), 0);
if (n > 0) {
    // 读到数据
} else if (n == 0) {
    // 连接关闭
} else {
    if (errno == EAGAIN || errno == EWOULDBLOCK) {
        // 当前无数据，稍后再试
    } else {
        // 真正的错误
        perror("recv error");
    }
}
```

非阻塞模式下，如果没有数据可读，`recv()` 立即返回 -1，并设置：`errno=EAGAIN` 或 `EWOULDBLOCK` 。

## 总结

阻塞与非阻塞模式是 socket 编程中两个核心的 I/O 模式，理解它们的行为差异是构建稳定、高性能网络程序的基础。阻塞模式简单直观，适用于连接数量少、逻辑简单的场景；而非阻塞模式虽然实现上更复杂，但结合 I/O 多路复用机制（如 epoll）则更适合高并发环境。

| 函数      | 阻塞模式行为             | 非阻塞模式行为（O_NONBLOCK）                          |
| ------- | ------------------ | -------------------------------------------- |
| connect | 等待连接成功或失败，可能阻塞较久   | 立即返回，返回 -1 并设置 errno = EINPROGRESS，需后续检测连接状态 |
| accept  | 没有新连接会阻塞           | 没有连接时立即返回 -1，errno = EAGAIN 或 EWOULDBLOCK    |
| recv    | 没有数据时阻塞直到数据到达或连接关闭 | 无数据时立即返回 -1，errno = EAGAIN                   |
| send    | 缓冲区满时会阻塞直到可以发送     | 缓冲区满时立即返回 -1，errno = EAGAIN                  |
