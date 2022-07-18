---
layout: post
title: 常见的二进制漏洞(二)-格式化字符串漏洞
date: 2022-06-24 15:55
categories: main
---

## 格式化输出函数说明

格式化字符串漏洞经常发生在printf等函数里
printf是一个变参函数，其中包含一个强制参数格式字符串和可选参数。

printf示例
```
常规printf
printf("Hello %%");             //"Hello %"
printf("Hello world!")          //"Hello world!"
printf("Number:%d", 123)        //"Number:123"
printf("%s %s", "Format", "Strings");   //"Format Strings"
printf("%12c", 'A');            //"           A"
printf("%16s", "Hello");        //"             Hello"

赋值printf
int n;
printf("%12c%n", 'A', &n);      //n = 12
printf("%16s%n", "Hello");      //n = 16

变参数位置printf
printf("%2$s %1$s", "Format", "Strings");   //"Strings Format"
```
## 格式化字符串漏洞
格式化字符串从防御手段分为两类，一种是检查参数个数，另一类则是安全机制FORTIFY_SOURCE对格式化字符串函数进行检查。
### 格式化字符串原理
根据调用约定，printf的格式字符串会依次读取栈的内容，当格式字符串的值多于可选参数时，就会对栈的内容进行读取，从而造成信息泄露。
简单示例
```
#include <stdio.h>
int main()
{
    printf("%s-%d-%x-%x", "Hello", 123)
    return 0;
}
```
通过OD调试，可以看出，%x在没有参数的情况下会将栈的内容输出
![OD调试](/assets/img/2.png)

### 漏洞利用

对于格式化字符串的漏洞利用主要有:程序崩溃、栈数据泄露、任意地址内存泄漏、栈数据覆盖、任意地址内存覆盖。

#### 使程序崩溃

能够触发程序崩溃的格式化字符串有以下三种类型
- 对于每一个%s,printf都要从栈中获取地址，并打印该地址所指向的内存，当出现空字符使会导致程序崩溃
- 从栈中获取的地址是非法的
- 从栈中获取的地址是被保护的

#### 栈数据泄露

通过格式化字符串 **%08x** 或者 **%p** 可获取栈内对应的地址，当格式化字符串多的时候就可以去获取栈内其他地址，一般为ebp和rop的值

#### 任意地址内存泄漏

当格式化字符串可以由自己控制时，便可以输入"%n$s",此时使argn的位置为%s的地址，便可以对该地址进行任意位置的读操作。

#### 栈数据覆盖


