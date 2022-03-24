---
title: Chapter-24 disassembly部分详解
description: Chapter 24 of FreeFlyOS
toc: true
authors:
tags:
weight: 24
categories:
series:
date: '2022-03-24'
lastmod: '2022-03-24'
draft: false
---
#### 该目录主要是可执行文件的反汇编

只能看到.text段的数据，简单示意下。

boot_disass.md  =====》》》》bootblock(引导程序)的反汇编

kernel_disass.md   =====》》》》kernel(内核)的反汇编代码

test_cat.md =====》》》》test_cat(cat测试程序)的反汇编代码

test_exec_disass.md =====》》》》test_exec(exec测试程序)的反汇编代码

test_pipe_disass.md =====》》》》test_pipe(pipe测试程序)的反汇编代码

通过以下命令可获得:

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-VIkwTsLq-1609683391784)(readme.assets/image-20201231143254582.png)\]](https://img-blog.csdnimg.cn/20210103221647465.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-f5wIkfAg-1609683391785)(readme.assets/image-20201231143309533.png)\]](https://img-blog.csdnimg.cn/20210103221657915.png)


![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-fFmJFKyE-1609683391786)(readme.assets/image-20201231143320631.png)\]](https://img-blog.csdnimg.cn/20210103221708746.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-plOMEAl7-1609683391787)(readme.assets/image-20201231143332349.png)\]](https://img-blog.csdnimg.cn/20210103221720384.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)



