

```c
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t count);  
```

函数从打开的设备或文件中读取数据。

返回值：成功返回读取的字节数，出错返回-1并设置 errno；如果在调用 read 之前已经达到文件末尾，则这次 read 返回0。

参数：
- buf 读上来的数据保存在缓冲区 buf 中，同时文件的当前读写位置向后移；
- count 是请求读取的字节数；