---
title: Chapter-20 test_exec部分详解
description: Chapter 20 of FreeFlyOS
toc: true
authors:
tags:
weight: 20
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


```

## test.c

测试exec是否能执行prog程序

```
#include "../kernel/user/stdio.h"
#include "../kernel/user/user_syscall.h"
#define NULL ((void *)0)
void main(int argc,char **argv){
    int arg_idx = 0; 
    while(arg_idx < argc) { 
        printf("argv[%d] is %s\n", arg_idx, argv[arg_idx]); 
        arg_idx++; 
    }
    printf("Nice to meet you! It's all! See You! Goodbye!\n");
    while(1);
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


