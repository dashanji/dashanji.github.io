---
title: Chapter-9 timer部分详解
description: Chapter 9 of FreeFlyOS
toc: true
authors:
tags:
weight: 9
categories:
series:
date: '2022-03-24'
lastmod: '2022-03-24'
draft: false
---
## timer.c

定时器初始化

```
#include "timer.h"
#include "../asm/asm.h"
#include "../pic/pic.h"
#include "../interrupt/trap.h"

unsigned int volatile jiffies; //记录自系统启动以来产生的节拍总数

unsigned int volatile second; //根据节拍总数换算成秒数

void timer_init(unsigned int frequency){
        //Intel 8253/8254 PIT芯片 I/O端口地址范围是40h-43h，输入频率为1193180，frequency为每秒中断次数
        unsigned int divisor=1193180/frequency;
        //将8253/8254芯片设置为模式3
        outb(0x43,0x36);
        unsigned char low=divisor&0xff;
        unsigned char high=(divisor>>8)&0xff;
        outb(0x40,low);
        outb(0x40,high);
        jiffies=0;   //初始化节拍总数，防止更换页表时初始化值更改
        second=0; //初始化秒数
        pic_enable(IRQ_TIMER);
}
```

## timer.h

```
#ifndef  _TIMER_H_
#define _TIMER_H_

void timer_init(unsigned int frequency);

#endif
```

定时器8253驱动原理如下，另外补充了FreeFlyOS的时钟每过1S中断一次，同时进行进程调度。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-YND0WC0D-1609680232206)(readme.assets/image-20210102223147461.png)\]](https://img-blog.csdnimg.cn/20210103212436166.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-j8fWBrMZ-1609680232208)(readme.assets/image-20210102223250050.png)\]](https://img-blog.csdnimg.cn/20210103212454522.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-YKjZlxfV-1609680232209)(readme.assets/image-20210102223302336.png)\]](https://img-blog.csdnimg.cn/20210103212509811.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-RpN25xYs-1609680232210)(readme.assets/image-20210102223314668.png)\]](https://img-blog.csdnimg.cn/20210103212523819.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-Pk2iNt22-1609680232211)(readme.assets/image-20210102223326120.png)\]](https://img-blog.csdnimg.cn/20210103212537359.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-nAXh8AzY-1609680232212)(readme.assets/image-20210102223339670.png)\]](https://img-blog.csdnimg.cn/20210103212549183.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-tIXtfY8s-1609680232213)(readme.assets/image-20210102223356663.png)\]](https://img-blog.csdnimg.cn/2021010321261039.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)

