---
title: Chapter-22 test_pipe部分详解
description: Chapter 22 of FreeFlyOS
toc: true
authors:
tags:
weight: 22
categories:
series:
date: '2022-03-24'
lastmod: '2022-03-24'
draft: false
---
## start.S

```
.text
.code32
.extern main
.extern exit
.global _start
_start:
    push %ebx
    push %ecx
    call main

    push %eax
    call exit
```



## test.c

测试管道用的程序

```
#include "../kernel/user/stdio.h"
#include "../kernel/user/user_syscall.h"
int main(int argc, char** argv) {
   int fd[2] = {-1};
   pipe(fd);
   int pid = fork();
   if(pid) {	  // 父进程
      close(fd[0]);  // 关闭输入
      write(fd[1], "Hi, my son, I love you!", 24);
      printf("\nI`m father, my pid is %d\n", user_sys_getpid());
      return 8;
   } else {
      close(fd[1]);  // 关闭输出
      char buf[32] = {0};
      read(fd[0], buf, 24);
      printf("\nI`m child, my pid is %d\n", user_sys_getpid());
      printf("I`m child, my father said to me: \"%s\"\n", buf);
      return 9;
   }
}
```



## test.ld

```
/* 
**   链接脚本
*/
OUTPUT_FORMAT(elf32-i386)
OUTPUT_ARCH(i386)
ENTRY(_start)
/*
*   ld有多种方法设置进程入口地址, 按以下顺序: (编号越前, 优先级越高)
*           1, ld命令行的-e选项
*           2, 连接脚本的ENTRY(SYMBOL)命令
*           3, 如果定义了start 符号, 使用start符号值
*           4, 如果存在 .text section , 使用.text section的第一字节的位置值
*           5, 使用值0
*
*
*/

SECTIONS 
{
    /* 将定位器符号置为0x7c00 */
    . = 0x80000000;

    .text : { *(.text) }

    .rodata : { *(.rodata) }
    .data : { *(.data) }

    .bss : { *(.bss) }
    
    
    
}
```


