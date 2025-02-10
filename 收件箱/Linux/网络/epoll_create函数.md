
Linux 网络编程中，很长的时间都在使用 select 来做事件触发。在新的 Linux 内核中，有了一种替换它的机制，就是 epoll。

因为在内核中的 select 实现，它是采用轮询来处理的，轮询的文件描述符数目越多，耗时越多。而 epoll 无须遍历整个被侦听的描述符集，只要遍历那些被内核 I/O 事件异步唤醒而加入 Ready 队列的描述符集合就行了。相对于 select 和 poll 来说，epoll 更加灵活，没有描述符的限制。

函数原型：
```c
int epoll_create(int size);
```

创建一个 epoll 句柄，size 用来告诉内核这个监听的数目一共有多大。当创建好 epoll 句柄后，它就是会占用一个 fd 值，所以在使用完 epoll 后，必须调用 close() 关闭，否则可能导致 fd 耗尽。