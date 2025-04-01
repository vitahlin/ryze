

# 测试

执行单个测试命令：
```shell
> ./runtest --single "unit/dump" --only "RESTORE can set LFU"
```

# 网络

##### getaddrinfo 函数

getaddrinfo 是一个用于解析主机名、服务名以及相关网络信息的函数，它被广泛用于网络编程中，尤其是当需要处理不同协议（如 IPv4 和 IPv6）时。getaddrinfo() 的一个重要特点是它能够根据系统的配置自动选择适当的地址族（例如 IPv4 或 IPv6），并为后续的套接字操作提供必要的地址信息。

```c
int getaddrinfo(const char *node, 
                const char *service, 
                const struct addrinfo *hints, 
                struct addrinfo **res);
```

- node: 主机名或 IP 地址。可以是：
    - 主机名（例如 “www.example.com”）。
    - IP 地址（例如 “192.168.1.1” 或 “2001:db8:: 1”）。
    - 如果为 NULL，则 getaddrinfo 会返回本地地址（使用 INADDR_ANY 或 IN6ADDR_ANY_INIT）。
- **service**: 服务名或端口号。可以是：
    - 服务名称（例如 “http” 或 “ftp”）。
    - 端口号（如 “80” 或 “443”）。
    - 如果为 NULL，则表示不需要解析端口。
- • **hints**: 一个指向 struct addrinfo 的指针，用于提供给 getaddrinfo 解析的提示（例如是否使用 IPv4 或 IPv6，是否使用流式套接字等）。可以根据需要进行设置，否则设置为全零结构，表示不限制。
- • **res**: 一个指向 struct addrinfo * 的指针，用于存储解析结果。getaddrinfo() 会将返回的地址信息以链表的形式存储在这个指针指向的区域。每个链表节点包含一个网络地址（例如 sockaddr_in 或 sockaddr_in6）。

- 成功时，返回 0，并且 res 会指向链表的第一个元素（存储了地址信息）。
- 失败时，返回一个非零值，具体的错误码可以通过 gai_strerror() 获取错误描述。
