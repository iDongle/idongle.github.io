---
layout: post
title: 常见的二进制漏洞(一)-整数溢出
date: 2022-06-24 8:30
categories: main
---

## 整数的范围
#### 32位
| 数据类型                   | 最小值      | 最大值     |
| -------------------------- | ----------- | ---------- |
| int(32bit 默认为signed)    | -2147483648 | 2147483647 |
| unsigned int               | 0           | 4294967295 |
| short (16bit 默认为signed) | -32768      | 32767      |
| unsigned short             | 0           | 65535      |
| long(32bit 默认为signed)   | -2147483648 | 2147483647 |
| unsigned long              | 0           | 4294967295 |


#### 64位
64位数据与32位数据大致相同，只有long和unsigned long和32位不同
| 数据类型                 | 最小值               | 最大值               |
| ------------------------ | -------------------- | -------------------- |
| long(64bit 默认为signed) | -9223372036854775808 | 9223372036854775807  |
| unsigned long            | 0                    | 18446744073709551615 |

## 整数异常的类型
整数的异常通常分为三种，溢出，回绕和截断

### 溢出
  溢出经常发正在有符号数相加或相减的时候，有符号数最高位为符号位，在进行运算时有可能改变符号位的值，产生溢出，影响OF标志位。
  溢出分为上溢出和下溢出两种
  ```
  int i = INT_MAX(2147483647);
  i = i + 1;        //上溢出
  printf("i = %d", i);
  ```
  最终产生的结果为-2147483648
  从二进制分析如下
  ```
   0111 1111 1111 1111 1111 1111 1111 1111
  +                                   0001
-------------------------------------------
   1000 0000 0000 0000 0000 0000 0000 0000(-2147483648)
   (7FFF FFFF + 0000 0001 = 8000 0000)
  ```
下溢出分析
  ```
  int i = INT_MIN(-2147483648);
  i = i - 1;        //下溢出
  printf("i = %d", i);
  ```
  最终产生的结果为2147483647
  从二进制分析如下
  ```
    1000 0000 0000 0000 0000 0000 0000 0000
  -                                    0001
--------------------------------------------     
    0111 1111 1111 1111 1111 1111 1111 1111(2147483647)
    (8000 0000 - 0000 0001 = 7FFF FFFF)
  ```

### 回绕
  回绕发生在无符号数相加或相减的时候，当最大值满足不了位数时，就会产生进位回绕行为，影响CF标志位。
  回绕也分为两种情况
  ```
  int i = INT_MAX(4294967295);
  i = i + 1;        //超出最大值，产生回绕，结果为0
  i = INT_MIN(0);
  i = i - 1;        //超过最小值，产生回绕，结果为4294967295;
  ```
  从二进制分析如下
```
     1111 1111 1111 1111 1111 1111 1111 1111
  +                                     0001
--------------------------------------------
 (1)0000 0000 0000 0000 00000 0000 0000 0000
```
```
 (1)0000 0000 0000 0000 0000 0000 0000 0000
  -                                    0001
--------------------------------------------
 (0)1111 1111 1111 1111 1111 1111 1111 1111

```
### 截断
截断常发生在不是同一种类型的数据进行运算时，将宽度较大的数强制转换为宽度较小的数，从而在高位发生截断。
例子如下
```
0xffffffff + 0x00000001
= 0x0000000100000000(long long)
= 0x00000000(long,截取低32位)
```
```
0x00123456 * 0x00654321
= 0x000007336BF94116(long long)
= 0x6BF94116(long,截取低32位)
```
## 整形溢出漏洞发生位置
整形溢出多发生于size_t作为参数的函数中
size_t是**无符号整数**类型的sizeof()的结果
void memcpy(void *dest, const void *src, size_t n);
char *strncpy(char *dest, const char *src, size_t n);
- 情形一
  当攻击者给长度赋值为负数时，sizeof()函数会将该值变为非常大的正数常量，从而可以复制大量的内容，造成缓冲区溢出。
  ```
  len = -1
  sizeof(len) = 4294967295;
  ```
- 情形二
  当有字符需求限制时，可以将长度设为非常大的值，在进行运算时，会发生回绕，从而使其符合字符需求限制，从而绕过检测，造成缓冲区溢出。
  ```
  len = 0xffffffff
  len + 5 = 0x00000004
  ```
## 案例

```
#include <stdio.h>
#include <string.h>

int main()
{
    char passwd_buf[11];
    unsigned char passwd_len = strlen(argv[1]);
    if(passwd_len >= 4 && passwd_len <= 8)
    {
        printf("Good Job");
        strcpy(passwd_buf, argv[1]);
    }
    else
    {
        printf("Wrong");
    }
    return 0;
}
```
可以看到passwd_len变量为unsigned char类型，该类型作为数值时的范围为1BYTE,取值范围为0-255，当长度大于该值时就会产生溢出，利用这点可以绕过passwd_len大于4小于8的限制

内存变量堆栈如图所示
```
-0000000000000020 ;
-0000000000000020
-0000000000000020 var_20          dq ?
-0000000000000018                 db ? ; undefined
-0000000000000017                 db ? ; undefined
-0000000000000016                 db ? ; undefined
-0000000000000015                 db ? ; undefined
-0000000000000014 var_14          dd ?
-0000000000000010                 db ? ; undefined
-000000000000000F                 db ? ; undefined
-000000000000000E                 db ? ; undefined
-000000000000000D                 db ? ; undefined
-000000000000000C dest            db 11 dup(?)
-0000000000000001 var_1           db ?
+0000000000000000  s              db 8 dup(?)
+0000000000000008  r              db 8 dup(?)
+0000000000000010
+0000000000000010 ; end of stack variables
```
其中var_1为passwd_len值
可以看出来溢出几个字节很容易覆盖原来ret变量的值
![例子图片](/assets/img/1.png)
从而达到缓冲区溢出，修改EIP的目的