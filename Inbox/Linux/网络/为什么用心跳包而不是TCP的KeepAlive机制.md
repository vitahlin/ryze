很多应用层协议都有心跳包机制，通常是客户端每隔一小段时间向服务器发送一个数据包，通知服务器自己仍然在线，并传输一些可能必要的数据。使用心跳包的典型协议是IM。

许多 IM 协议实现了自己的心跳机制，而不是直接依赖于底层的 KeppAlive 机制。

# 原因

自己实现的心跳机制通用，可以无视底层的 UDP 或者 TCP 协议。如果只是用 TCP 协议的话，那么直接使用 KeepAlive 机制就足够了。

心跳除了说明应用程序还活着（进程还在，网络通畅），更重要的是表明应用程序还能正常工作。而 TCP KeepAlive 由操作系统负责探查，即使进程死锁或阻塞，操作系统也会如常收发 TCP KeepAlive 消息，对方无法得知这一异常——《Linux 多线程服务端编程》。

KeepAlive 机制很多情况无法检测出来，如网络连接被软件禁用等，不够可靠，网络状态复杂的情况下这种情况尤其严重。

自己实现心跳可以加入更灵活与实用的机制，比如少了一个心跳，可以马上再次检查，检查间隔递减，这样可以更快的感知网络状态，而不是等待固定时间。