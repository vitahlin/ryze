---
title: struct声明中的__attribute__ ((__packed__))关键字
slug: unpacking-the-attribute-packed-in-struct
date: 2024-09-03
categories:
  - Lang
tags:
  - C
featuredImage: https://sonder.vitah.me/featured/3baa8e7de9911154e1f7d50bf98b854b.webp
desc: 介绍了如何通过使用__attribute__ ((__packed__))关键字来取消默认的字节对齐，从而减少内存占用。通过一个简单的代码示例展示了在字节对齐和非字节对齐情况下结构体的内存占用对比。
---

C/C++中，建立一个结构体的时候，会进行字节对齐操作，所以往往比实际变量占用的字节数要多一些。

但是当我们不想要字节对齐的时候，有没有办法取消字节对齐？答案是可以，就是在结构体声明当中，加上 `__attribute__ ((__packed__))` 关键字，它可以让结构体以紧凑排列的形式，减少内存占用，在 Redis 大量使用，如：
```c
struct __attribute__ ((__packed__)) sdshdr5 {  
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */  
    char buf[];  
};

struct __attribute__ ((__packed__)) sdshdr8 {  
    uint8_t len; /* used */  
    uint8_t alloc; /* excluding the header and null terminator */  
    unsigned char flags; /* 3 lsb of type, 5 unused bits */  
    char buf[];  
};
```

可以通过示例代码来验证：
```c
#include <stdio.h>
#include <iostream>
 
using namespace std;
 
struct test1 {
    char c;
    int i;
};
 
struct __attribute__ ((__packed__)) test2 {
    char c;
    int i;
};
 
int main()
{
    cout << "size of test1:" << sizeof(struct test1) << endl;
    cout << "size of test2:" << sizeof(struct test2) << endl;
}
```

运行结果：
```shell
size of test1:8
size of test2:5
```

显而易见，test1 结构体里面没有加关键字，它采用了4字节对齐的方式，即使是一个 char 变量，也占用了4字节内存，int 占用4字节，共占用了8字节内存，这在64位机器当中将会更大。

而 test2 结构体，再加上关键字之后，结构体内的变量采用内存紧凑的方式排列，char 类型占用1字节，int 占用4字节，总共占用了5个字节的内存。