---
layout: post
title: 关于个人的一些说明
date: 2022-06-20 8:30
categories: main
---

## Static Canaries
&nbsp;&nbsp;Static Canaries是用于对抗栈溢出攻击的技术，即SSP安全机制，有时也叫做Stack cookies。<p>
&nbsp;&nbsp;Canary的值是栈上的一个随机数，在程序启动时随机生成并保存在比返回地址更低的位置。由于栈溢出是从低地址向高地址覆盖，因此，攻击者在进行栈攻击时，一定会先覆盖掉Canary，所以通过检查该值可检查栈是否被篡改。

## Linux开启/关闭Canaries保护
gcc -fstack-protector canary.c -o f.out <span style="text-align:right">#开启</span>
<p>
gcc -fno-stack-protector canary.c -o f.out    #关闭

## 例子
```
#include <stdio.h>
int main()
{
  char buf[10];
  scanf("%s",buf);
  return 0;
}
```
