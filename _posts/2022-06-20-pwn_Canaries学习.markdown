---
layout: post
title: Linux安全机制说明
date: 2022-06-20 8:30
categories: main
---

# Linux安全机制
通过checksec命令可查看文件的保护类型
- checksec --file=f.out
安全机制大概分为5种

| RELRO         | STACK CANARY | NX        | PIE         | FORTIFY |
| ------------- | ------------ | --------- | ----------- | ------- |
| Partial RELRO | Canary found | NX enable | PIE enabled | No      |

## Static Canaries
&nbsp;&nbsp;&nbsp;&nbsp;Static Canaries是用于对抗栈溢出攻击的技术，即SSP安全机制，有时也叫做Stack cookies。<p>
&nbsp;&nbsp;&nbsp;&nbsp;Canary的值是栈上的一个随机数，在程序启动时随机生成并保存在比返回地址更低的位置。由于栈溢出是从低地址向高地址覆盖，因此，攻击者在进行栈攻击时，一定会先覆盖掉Canary，所以通过检查该值可检查栈是否被篡改。

### 开启/关闭Canaries保护
```
gcc -fstack-protector canary.c -o f.out        #开启
gcc -fno-stack-protector canary.c -o fn.out    #关闭
```
### 例子
```
#include <stdio.h>
int main()
{
  char buf[10];
  scanf("%s",buf);
  return 0;
}
```
可以看到被canary保护的程序在溢出后无法正常执行程序
<p>
```

└─# ./f.out 
11111111111111111111111111111111111111111111
*** stack smashing detected ***: terminated
zsh: IOT instruction  ./f.out

```
没被保护的程序则出现了栈溢出报错
```
└─# ./fn.out 
11111111111111111111111111111111
zsh: segmentation fault  ./fn.out
```

通过gdb查看有canary的反汇编情况
```
(gdb) disassemble
Dump of assembler code for function main:
   0x0000555555555149 <+0>:     push   %rbp
   0x000055555555514a <+1>:     mov    %rsp,%rbp
   0x000055555555514d <+4>:     sub    $0x20,%rsp
=> 0x0000555555555151 <+8>:     mov    %fs:0x28,%rax
   0x000055555555515a <+17>:    mov    %rax,-0x8(%rbp)
   0x000055555555515e <+21>:    xor    %eax,%eax
   0x0000555555555160 <+23>:    lea    -0x12(%rbp),%rax
   0x0000555555555164 <+27>:    mov    %rax,%rsi
   0x0000555555555167 <+30>:    lea    0xe96(%rip),%rax        # 0x555555556004
   0x000055555555516e <+37>:    mov    %rax,%rdi
   0x0000555555555171 <+40>:    mov    $0x0,%eax
   0x0000555555555176 <+45>:    call   0x555555555040 <__isoc99_scanf@plt>
   0x000055555555517b <+50>:    mov    $0x0,%eax
   0x0000555555555180 <+55>:    mov    -0x8(%rbp),%rdx
   0x0000555555555184 <+59>:    sub    %fs:0x28,%rdx
   0x000055555555518d <+68>:    je     0x555555555194 <main+75>
   0x000055555555518f <+70>:    call   0x555555555030 <__stack_chk_fail@plt>
   0x0000555555555194 <+75>:    leave  
   0x0000555555555195 <+76>:    ret
End of assembler dump.
```


查看没有canary保护的程序
```
(gdb) disassemble 
Dump of assembler code for function main:
   0x0000555555555139 <+0>:     push   %rbp
   0x000055555555513a <+1>:     mov    %rsp,%rbp
=> 0x000055555555513d <+4>:     sub    $0x10,%rsp
   0x0000555555555141 <+8>:     lea    -0xa(%rbp),%rax
   0x0000555555555145 <+12>:    mov    %rax,%rsi
   0x0000555555555148 <+15>:    lea    0xeb5(%rip),%rax        # 0x555555556004
   0x000055555555514f <+22>:    mov    %rax,%rdi
   0x0000555555555152 <+25>:    mov    $0x0,%eax
   0x0000555555555157 <+30>:    call   0x555555555030 <__isoc99_scanf@plt>
   0x000055555555515c <+35>:    mov    $0x0,%eax
   0x0000555555555161 <+40>:    leave  
   0x0000555555555162 <+41>:    ret    
End of assembler dump.
```

可以看出区别仅在于在栈位置rbp-8的地方添加了一个fs:[0x28]的值，该值为系统TCB(线程控制块)结构体里的一个字段。其内容根据需求为随机值。

## No-eXecute
&nbsp;&nbsp;&nbsp;&nbsp;No-eXecute(NX)，表示不可执行。其原理是将数据所在的内存页(比如栈，堆，.data区)标识为不可执行，如果程序产生溢出转入执行shellcode时，cpu就会抛出异常。在windows上被成为DEP(数据执行保护)。

### 开启/关闭NX

```
gcc -z noexecstack hello.c        #开启
gcc -z execstack hello.c          #禁用
```
### 例子
```
#include <unistd.h>

void vlu_fun()
{
  char buf[128];
  read(STDIN_FILEND, buf, 256);
}
int main()
{
  vlu_fun();
  write(STDOUT_FILENO, "Hello world!\n", 13);
  return 0;
}
```

## ASLR和PIE
&nbsp;&nbsp;&nbsp;&nbsp;ASLR(Address Space Layout Randomization,地址空间布局随机化)是将内存布局随机化。攻击者在攻击程序时，往往知道内存布局，从而可以知道shellcode和其他代码的位置，在此基础上，将地址随机化可以将漏洞利用的难度提高。
&nbsp;&nbsp;&nbsp;&nbsp;在Linux系统中，ASLR的配置在/proc/sys/kernel/randomize_va_space中，其中0表示关闭ASLR，1表示部分开启，2表示完全开启。
<p>
&nbsp;&nbsp;&nbsp;&nbsp;ASLR是系统层面的地址随机，其中随机话的有Executable,PLT,Heap,Stack和Shared Library。在此基础上，攻击者任然可以通过ret2plt,GOT劫持，爆破手段来绕过。PIE(Position-Independebt Executable)技术的出现是将程序内部引入无关的可执行代码，使程序加载到任意位置。将ASLR和PIE技术相结合，就可以使攻击者对内存布局一无所知。

### 开启/关闭ASLR/PIE

开启/关闭ASLR只需在/proc/sys/kernel/randomize_va_space中修改数值即可
<p>
开启/关闭PIE
```
gcc -pie -fno-pie test.c        #开启
gcc -no-pie test.c              #关闭
```

## FORTIFY_SOURCE
&nbsp;&nbsp;&nbsp;&nbsp;缓冲区溢出经常发生在危险函数的地方，FORTIFY_SOURCE的出现就是为了检查危险函数，在编译时确定危险是否存在，或者将函数替换为比较安全的函数，从而降低缓冲区溢出风险的发生。
&nbsp;&nbsp;&nbsp;&nbsp;检查的函数例如memcpy, mempcpy, memmove, memset, strcpy, stpcpy, strncpy, strcat, strncat, sprintf, vsprintf, snprintf, vsnprintf, gets等，目的是检查dest变量内存是否溢出

### 开启/关闭FORTIFY_SOURCE
默认情况下，在某些系统上该机制时关闭的。开启需要在编译时手动指定等级。
```
-D_FORIFY_SOURCE=1,开启缓冲区溢出攻击检查
-D_FORITY_SOURCE=2,开启缓冲区溢出攻击检查和格式化字符串攻击检查
```


## RELRD
&nbsp;&nbsp;&nbsp;&nbsp;该安全机制的出现是为了解决Linux程序延迟绑定的问题。