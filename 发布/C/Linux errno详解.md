---
title: Linux errno详解
slug: linux-errno-explanation
categories:
  - Lang
tags:
  - C
featuredImage: https://sonder.vitah.me/ryze/9426c537e3689e348ac2eb059d2b4777.webp
desc: 在Linux编程中，errno 是一个至关重要的全局变量，它用于表示系统调用或库函数执行失败时的具体错误类型。在本篇博客中，我们将深入解析 errno 的工作原理，常见错误码及其含义，并提供示例代码来帮助开发者更好地理解如何处理错误信息。无论是文件操作、网络编程，还是进程管理，正确使用 errno 都能帮助你更快地定位问题，提高程序的健壮性。
draft: false
---

## errno 介绍

Linux中系统调用的错误通过函数返回值来表示，并通过特殊变量 `errno` 来描述。错误都存储于 `errno` 中，`errno ` 是一个全局变量（在 `<errno.h>` 头文件中定义），由操作系统维护，用于存储最近一次系统调用或库函数出错时的错误代码，即下一次的错误码会覆盖掉上一次的错误。

`errno` 是一个由POSIX和ISO C标准定义的符号，看起来就像是一个整形变量。当**系统调用或者库函数发生错误的时候，它的值将会被改变。**

使用 `error` 要注意三点：
1. 如果系统调用或库函数执行正确的时候，`errno` 的值不会被置0。执行函数A的时候发生了错误，`errno` 被改变，接下来执行函数B，如果函数B正确执行，`errno` 还保留函数A发生错误时被设置的值。
2. 系统调用或库函数正确执行，并不保证 `errno` 的值不会被改变
3. 任何错误号都是非0的

所以，当需要用 `errno` 来判断函数是否正确执行的时候，最好先将 `errno` 清0，函数执行结束时，通过其返回值判断函数是否正确执行，若没有正确执行，再根据 errno 判断哪里发生了错误。

## 如何使用 errno

当某些系统调用或标准库函数执行失败时，它们通常返回一个错误值（例如 `-1` 或 `NULL` ），并且会设置 `errno` 来指示具体的错误类型。例如：
```c
#include <stdio.h>
#include <errno.h>
#include <string.h>

int main() {
    FILE *fp = fopen("non_existent_file.txt", "r");
    if (fp == NULL) {
        printf("打开文件失败，错误码: %d\n", errno);
        printf("错误信息: %s\n", strerror(errno));
    }
    return 0;
}
```

## 打印错误信息函数

### perror

```c
#include <stdio.h>
void perror(const char *s);
```

它先打印 `s` 指向的字符串，然后输出当前 ` errno ` 值所对应的错误提示信息，例如当前 ` errno ` 若为12，调用 ` perror("ABC") `，会输出 `ABC: Cannot allocate memory` 。

### strerror

```c
#include <string.h>
char *strerror(int errnum);
```

它返回 `errnum` 的值所对应的错误提示信息，例如 `errnum` 等于12的话，它就会返回 `Cannot allocate memory` 。