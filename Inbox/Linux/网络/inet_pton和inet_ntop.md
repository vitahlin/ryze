
### inet_pton函数

将 IPv4 和 IPv6 地址从点分十进制转换为二进制。
```c
#include <arpa/inet.h>
int inet_pton(int af, const char *src, void *dst);
```

参数：

-   `af` 地址族类型，`AF_INET` 或者 `AF_INET6`
-   `src` 字符型地址，如 `xxx.xxx.xxx.xxx`
-   `dst` 函数将地址转换为 `in_addr` 的结构体，并复制在 `*dst` 中

返回值：如果函数出错返回一个负值。如果参数 `af` 指定的地址族和 `src` 格式不对，函数将返回0。

### inet_ntop函数

与函数 inet_ntop 相反，该函数是将IP地址从二进制转换为点分十进制。
```c
#include <arpa/inet.h>
const char *inet_ntop(int af, const void *src, char *dst, socklen_t size);
```

参数：

-   af 地址族
-   src 二进制数组，类型为 struct in_addr
-   dst 保存转换后的点分十进制地址
-   size 所指向缓存区 dst 的大小，避免溢出，如果缓存区太小无法存储地址的值，则返回一个空指针

返回值：成功，返回字符串的首地址，错误返回NULL

### 示例

```c
#include <stdio.h>
#include <netinet/in.h>
#include <arpa/inet.h>

int main(int argc, char **argv) {
    char ip[20];
    struct in_addr s;

    printf("input IP address:");
    scanf("%s", ip);

    // 转换
    inet_pton(AF_INET, ip, &s);
    printf("inet_pton: 0x%x\n", s.s_addr);

    // 反转换
    inet_ntop(AF_INET, &s, ip, 16);
    printf("inet_ntop: %s\n", ip);

    return 0;
}
```

运行结果：
```shell
input IP address:192.168.3.1
inet_pton: 0x103a8c0
inet_ntop: 192.168.3.1
```