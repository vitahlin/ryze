
```c
#include<sys/socket.h>
int bind ( int sockfd, struct sockaddr * addr, socklen_t addrlen )
```

参数：
-   sockfd 指定地址与哪个套接字绑定，这是一个由之前的 socket 函数调用返回的套接字。调用 bind 函数之后，该套接字与一个相应的地址关联，发送到这个地址的数据可以通过这个套接字来读取与使用
-   addr 已经经过填写的有效的地址结构
-   addrlen 地址长度

返回值：成功返回0，失败返回-1


`bind` 函数并不是总需要调用，只要用户进程想与一个具体的地址或端口相关联的时候才需要调用这个函数。如果用户进程没有调用，程序可以依赖内核的自动选址机制来完成自动地址选择。一般情况下，对服务器进程需要调用 `bind` 函数，对客户进程则不需要调用 `bind` 函数。

示例代码：
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

对于IPv4来说，通配地址通常由 `INADDR_ANY` 来指定，其值一般为0.0.0.0，这个地址事实上表示不确定地址，或者“所有地址”，“任意地址”。