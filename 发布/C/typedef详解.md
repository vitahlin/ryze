---
title: typedef详解
slug: typedef-detailed-explanation
categories:
  - Lang
tags:
  - C
featuredImage: https://sonder.vitah.me/ryze/99e77209a0dbf2d3d43bdc409fb0fde8.webp
desc: 在 C 语言中，typedef 关键字用于为已有类型创建新的别名，从而提高代码的可读性和可维护性。本篇博客将深入解析 typedef 的用法，包括结构体重命名、指针类型定义、函数指针的应用，以及 typedef 和 struct 的区别，帮助你全面掌握 typedef 的使用技巧。
---

概述：是一种在计算机编程语言中用来声明自定义数据类型，配合各种原有数据类型来达到简化编程的目的的类型定义关键字。

用法：
```c
typedef int elemtype;
```

此时`elemtype`等价于`int`，即可用 `elemtype a;` 语句定义一个`int`类型的数据。


重点在于与结构体结合使用，例如：
```c
typedef struct tagMyStruct
{ 
    int iNum;
    long lLength;
} MyStruct;
```

这语句实际上完成两个操作：
1. 定义一个新的结构类型
```c
struct tagMyStruct
{ 
    int iNum; 
    long lLength; 
};
```

分析：`tagMyStruct` 称为“tag”，即“标签”，实际上是一个临时名字，`struct`  关键字和 `tagMyStruct` 一起，构成了这个结构类型，不论是否有 `typedef`，这个结构都存在。
**我们可以用 `struct tagMyStruct varName` 来定义变量，但要注意，使用 `tagMyStruct varName` 来定义变量是不对的，因为 `struct` 和 `tagMyStruct` 合在一起才能表示一个结构类型。** 

2. `typedef` 为这个新的结构起了一个名字，叫 `MyStruct`：
```c
typedef struct tagMyStruct MyStruct;
```

因此，`MyStruct` 实际上相当于 `struct tagMyStruct`，我们可以使用 `MyStruct varName` 来定义变量。
当然，MyStruct 可以命名为 tagMyStruct一样，例如 Redis 中的用法：
```c
typedef struct aeApiState {  
    int kqfd;  
    struct kevent *events;  
    char *eventsMask;   
} aeApiState;
```
`aeApiData` 既是**结构体标签**，也是 `typedef` 之后的类型名。

同样 `typedef` 也可以应用于 函数指针和 二级指针（指针的指针），如：
```c
typedef void (*FuncPtr)(int);  // 定义函数指针类型

void printNumber(int n) {
    printf("Number: %d\n", n);
}

int main() {
    FuncPtr func = printNumber;  // 等价于 void (*func)(int) = printNumber;
    func(100);
    return 0;
}
```

```c
typedef char** CharPtrPtr;  // 定义二级指针类型

int main() {
    char *str = "Hello";
    CharPtrPtr pptr = &str;  // 等价于 char** pptr = &str;
    printf("%s\n", *pptr);
    return 0;
}
```

| 方式    | typedef 形式                    | 等价写法               |
| ----- | ----------------------------- | ------------------ |
| 指针别名  | typedef int* IntPtr;          | int *p;            |
| 结构体指针 | typedef Point* PointPtr;      | Point *p;          |
| 函数指针  | typedef void (*FuncPtr)(int); | void (*func)(int); |
| 二级指针  | typedef char** CharPtrPtr;    | char **p;          |
| 链表指针  | typedef Node* NodePtr;        | Node *p;           |
使用 `typedef` 定义指针类型，可以让代码更易读，减少复杂指针声明带来的混淆。