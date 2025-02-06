
TCP客户端用 connect 函数来建立与 TCP 服务器的连接。

```c
#include<sys/socket.h>
int connect (int sockfd, const struct sockaddr * serv_addr, socklen_t addrlen);
```

-   sockfd 由 socket() 函数返回的套接字描述符
-   serv_addr 套接字地址结构的指针。套接字地址结构必须含义服务器的IP地址和端口号。
-   addrlen 套接字地址结构的大小

客户端在调用函数 connect 前不必非得调用 bind 函数，因为如果需要的话，内核会确定源IP地址，并选择一个临时端口作为源端口。

如果是TCP套接字，调用 connect 函数将激发TCP的三路握手过程，而且仅在连接建立成功或出错时才返回，其中出错返回可能由以下几种情况。

1.  客户端没有收到SYN的响应，返回ETIMEDOUT错误，表示连接不可达。例如指定一个并不存在的IP地址。
2.  对客户的响应是RST（表示复位），则表示服务器在指定的端口上没有进程等待连接（例如服务器进程没有在运行）。客户端一收到RST返回 ECONNREFUSED 错误。
3.  客户端发出的SYN在某个路由器上面引发了一个“**目的地不可达**” ICMP 错误。例如，指定一个因特网中不可到达的IP地址。