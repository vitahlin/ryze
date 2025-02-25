### 网络中如何唯一标识一个进程

在本地我们可以通过进程 PID 来唯一标识一个进程，在网络中，TCP/IP 协议族则用 IP 地址来唯一标识网络中的主机，而传输层的“协议+端口”标识主机中的应用程序（进程）。这样利用三元组（IP 地址，协议，端口）就可以标识网络的进程。

### 什么是socket

网络中的进程是通过 socket 来通信的，那么什么是 socket？socket 起源于 Unix，而 Unix/Linux 基本这些就是一切皆文件，都可以用 “打开 open->读写 read/write->关闭 close”来操作。socket 即是一种特殊的文件，一些 socket 函数就是对其进行的操作。

### socket函数

函数原型：
```c
#include <sys/types.h>          
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
```

当我们调用 socket 创建一个 socket 时，返回的 socket 描述字它存在于协议族（address family，AF_XXX）空间中，但没有一个具体的地址。如果想要给它赋值一个地址，就必须调用 bind()函数，否则就当调用 connect()、listen()时系统会自动随机分配一个端口。

#### 参数

-   domain

地址族，`domain` 用来设置网络通信的域名，也就是 IP 地址类型，常用的有 AF_INET 和 AF_INET6，AF_INET 表示 IPv4地址，AF_INET6 表示 IPv6地址。

-   type

type 为数据传输方式/套接字类型，常用的有 SOCK_STREAM（流格式套接字/面向连接的套接字） 和 SOCK_DGRAM（数据报套接字/无连接的套接字）。

-   protocol

protocol 表示传输协议，常用的有 IPPROTO_TCP 和 IPPTOTO_UDP，分别表示 TCP 传输协议和 UDP 传输协议。可以将 protocol 的值设为 0，系统会自动推演出应该使用什么协议。

#### 返回值

函数 `socket()` 并不总是执行成功，有可能会出现错误，错误的产生有多种原因，可以通过 errno 获得。

### 示例
```c
// 创建tcp套接字
int tcp_socket = socket(AF_INET, SOCK_STREAM, 0);  

// 创建unp套接字
int udp_socket = socket(AF_INET, SOCK_DGRAM, 0);
```