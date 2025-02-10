`select` 函数用来检查套接字字符是否已经准备好读写，提供了一种同时检查多个套接字的方法。

`select` 函数监视的文件描述符分3类，分别是 writefds、readfds、和 exceptfds。调用后 `select` 函数会阻塞，直到有描述符就绪（有数据可读、可写、或者有 except），或者超时（timeout 指定等待时间，如果立即返回设为 `null` 即可），函数返回。当 `select` 函数返回后，可以通过遍历 `fdset`，来找到就绪的描述符。

# 函数原型

函数原型：
```c
#include <sys/select.h>
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>
int select(int maxfdp, fd_set *readset, fd_set *writeset, fd_set *exceptset,struct timeval *timeout)
```

参数说明：

1.  `maxfdp`，被监听的文件描述符总数，它比所有文件描述符集合中的文件描述符的最大值大1，因为文件描述符是从0开始计数的；
2.  `readset` 可选参数，指向可读的描述符集合
3.  `writeset` 可选参数，指向可写的描述符集合
4.  `exceptset` 可选参数，指向异常的描述符集合
5.  `timeout` 用于设置 `select` 函数的超时时间，即告诉内核 `select` 等待多长时间之后就放弃等待，`timeout=NULL` 表示等待无限长时间`timeout` 参数有3种可能
    1. 永远等待，仅在一个描述符准备好I/O时才返回。这是 `timeout` 为空指针
    2. 根本不等待。检查描述符后立即返回，这称为轮询。此时，参数为 `timeval` 指针，其中的定时器值（指定的秒数和微秒数）必须为0
    3. 等待一个固定时间。由 `timeval` 指定秒数和微妙数

`timeval` 结构体如下：
```c
struct timeval
{      
    long tv_sec;   /*秒 */
    long tv_usec;  /*微秒 */   
};
```

返回值：
超时返回0；失败返回-1，具体错误内容可以查看 `errno`；成功返回大于0的整数，这个整数表示就绪描述符的数量。

  

# 常用的四个宏

在使用 `select` 函数时，会经常用到四个宏
```c
int FD_ZERO(int fd, fd_set *fdset);   
int FD_CLR(int fd, fd_set *fdset);  
int FD_SET(int fd, fd_set *fd_set);   
int FD_ISSET(int fd, fd_set *fdset);
```

`FD_SET`：将一个指定的文件描述符加入集合
`FD_CLR`：将一个指定的文件描述符从集合中删除
`FD_ISSET`：检查一个指定的文件描述符是否在集合中
`FD_ZERO`：清空集合