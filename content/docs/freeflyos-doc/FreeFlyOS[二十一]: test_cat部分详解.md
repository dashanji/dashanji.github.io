---
title: Chapter-21 test_cat部分详解
description: Chapter 21 of FreeFlyOS
toc: true
authors:
tags:
weight: 21
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

cat命令实现，属于外部命令

```
#include "../kernel/user/stdio.h"
#include "../kernel/user/user_syscall.h"
#include "../kernel/user/string.h"
#define NULL ((void *)0)
void main(int argc,char **argv){
    /*printf("hello nice to meet you\n");
    while(1);*/
   if (argc > 2 || argc == 1) {
      printf("cat: only support 1 argument.\neg: cat filename\n");
      exit(-2);
   }
   int buf_size = 1024;
   char abs_path[512] = {0};
   void* buf = malloc(buf_size);
   if (buf == NULL) { 
      printf("cat: malloc memory failed\n");
      return -1;
   }
   if (argv[1][0] != '/') {
      getcwd(abs_path, 512);
      user_strcat(abs_path, "/");
      user_strcat(abs_path, argv[1]);
   } else {
      user_strcpy(abs_path, argv[1]);
   }
   int fd = open(abs_path, O_RDONLY);
   if (fd == -1) { 
      printf("cat: open: open %s failed\n", argv[1]);
      return -1;
   }
   int read_bytes= 0;
   while (1) {
      read_bytes = read(fd, buf, buf_size);
      if (read_bytes == -1) {
         break;
      }
      write(1, buf, read_bytes);
   }
   free(buf,buf_size);
   close(fd);
   return 66;
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


