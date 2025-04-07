---
title: 
slug: 
categories: 
tags: 
featuredImage: 
desc: 
draft: true
---

什么是阻塞 socket，什么是非阻塞 socket。对于这个问题，我们要先弄清什么是阻塞/非阻塞。阻塞与非阻塞是对一个文件描述符指定的文件或设备的两种工作方式。
- 阻塞模式，就当某个函数“执行成功的条件”当前不能满足时，该函数会阻塞当前执行线程，程序执行流在超时时间到达或“执行成功的条件”满足后恢复继续执行。比如当试图对该文件描述符进行读写时，如果当时没有东西可读或者暂时不可写，程序就进入等待状态，直到有东西可读或者可写为止。
- 非阻塞模式恰恰相反，即使某个函数的“执行成功的条件”不当前不能满足，该函数也不会阻塞当前执行线程，而是立即返回，继续运行执行程序流。当没有东西可读或者不可写时，读写函数就马上返回，而不会等待。

阻塞和非阻塞模式下，我们常讨论的具有不同行为表现的 socket 函数一般有如下几个：
- connect
- accept
- send (Linux 平台上对 socket 进行操作时也包括 **write** 函数，下文中对 send 函数的讨论也适用于 **write** 函数)
- recv (Linux 平台上对 socket 进行操作时也包括 **read** 函数，下文中对 recv 函数的讨论也适用于 **read** 函数)

无论是 Windows 还是 Linux 平台，默认创建的 socket 都是阻塞模式的。不过我们可以通过函数 `fcntl()` 给创建的 socket 增加 **O_NONBLOCK** 标志来将 socket 设置成非阻塞模式。

fcntl 是一个在 Unix/Linux 系统中用于操作文件描述符的系统调用函数，定义在 <fcntl.h> 头文件中。它常用于：
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

示例代码，设置非阻塞模式：
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

### 建立连接 connect

默认行为，阻塞情况下，connect 首先发送 SYN 请求到服务器，当客户端收到服务器返回的 SYN 的确认时，则 connect 返回，否则的话一直阻塞。
```c
int sockfd = socket(AF_INET, SOCK_STREAM, 0);
connect(sockfd, (struct sockaddr *)&addr, sizeof(addr)); // 阻塞调用
```

上述代码会**阻塞等待连接完成**。如果服务器很慢，或者网络有问题，**程序就会卡在这里直到连接成功或失败（超时）**，适合简单、同步式逻辑。

非阻塞模式下，connect将启用 [TCP协议](https://zhida.zhihu.com/search?content_id=237420436&content_type=Article&match_order=1&q=TCP%E5%8D%8F%E8%AE%AE&zhida_source=entity)的[三次握手](https://zhida.zhihu.com/search?content_id=237420436&content_type=Article&match_order=1&q=%E4%B8%89%E6%AC%A1%E6%8F%A1%E6%89%8B&zhida_source=entity)，但是connect函数并不等待连接建立好才返回，而是立即返回，返回的错误码为[EINPROGRESS](https://zhida.zhihu.com/search?content_id=237420436&content_type=Article&match_order=1&q=EINPROGRESS&zhida_source=entity),表示正在进行某种过程。

```c
int flags = fcntl(sockfd, F_GETFL, 0);
fcntl(sockfd, F_SETFL, flags | O_NONBLOCK);

int ret = connect(sockfd, (struct sockaddr *)&addr, sizeof(addr));
if (ret < 0 && errno == EINPROGRESS) {
    // 正在连接中，不是错误！
    // 后续你可以用 select()/poll()/epoll() 等等待写事件
}
```

connect() 会**立即返回**，不会阻塞程序。
- 如果连接正在建立，会返回 -1，并设置 errno = EINPROGRESS。
- 后续需要使用 select()、poll() 或 epoll() 等 **I/O 多路复用机制**来等待连接完成。
- 连接成功后，socket 会变得可写；可以用 getsockopt() 判断是否成功。