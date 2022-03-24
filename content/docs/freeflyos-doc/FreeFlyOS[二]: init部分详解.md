---
title: Chapter-2 init部分详解
description: Chapter 2 of FreeFlyOS
toc: true
authors:
tags:
weight: 2
categories:
series:
date: '2022-03-24'
lastmod: '2022-03-24'
draft: false
---
### init.c

```
/*
*     init.c : 建立临时页表，开启分页
*/
#include "init.h"
#include "../mem/memlayout.h"
#include "../debug/debug.h"
/*
*        具体映射关系已经在kernel.ld中定义即，LMA=VMA-0xC0000000，故
*               只需将内核所在的地址写入页表，并开启分页即可。
*/
extern unsigned int kernel_end;
void init()
{
    //内核 起始页在页目录表中的第几项
    unsigned int kernel_pdt_idx=(KERNEL_START&page_mask)/(vmm_page_size*page_table_size);
    //内核栈 起始页在页目录表中的第几项
    unsigned int stack_pdt_idx=((KERNEL_STACK_START-KERNEL_STACK_SIZE)&page_mask)/(vmm_page_size*page_table_size);
    //user 
    //unsigned int user_pdt_idx=((unsigned int)0x800000&page_mask)/(vmm_page_size*page_table_size);

    //在页目录表项中设置对应的页表地址
    //init部分对应的页目录表项
    pdt[0]=(unsigned int)pt_init|vmm_page_present|vmm_page_rw|vmm_page_kernel;
    //C0000000开始的4MB对应的页目录表项
    pdt[kernel_pdt_idx-4]=(unsigned int)pt|vmm_page_present|vmm_page_rw|vmm_page_kernel;
    //内核部分对应的页目录表项
    pdt[kernel_pdt_idx]=(unsigned int)pt1|vmm_page_present|vmm_page_rw|vmm_page_kernel;
    pdt[kernel_pdt_idx+1]=(unsigned int)pt2|vmm_page_present|vmm_page_rw|vmm_page_kernel;
    pdt[kernel_pdt_idx+2]=(unsigned int)pt3|vmm_page_present|vmm_page_rw|vmm_page_kernel;
    //内核栈部分对应的页目录表项
    pdt[stack_pdt_idx]=(unsigned int)stack_pt|vmm_page_present|vmm_page_rw|vmm_page_kernel;
    //user部分对应的页目录项
    //pdt[user_pdt_idx]=(unsigned int)user_pt|vmm_page_present|vmm_page_rw|vmm_page_kernel;
    /*目前需要映射页表的只有三个部分，分别是init部分（一个页表）、内核部分（两个页表）、内核栈（两页）*/
    
    //因为在init中开启分页后，代码仍然是停留在init部分，所以需要将其虚拟地址映射到物理地址，显然这部分不会超过4MB
    //故将其虚拟地址0-4MB全部映射到物理内存0-4MB
    for(unsigned int i=0;i<page_table_size;i++){
        pt_init[i]=(i<<12)|vmm_page_present|vmm_page_rw|vmm_page_kernel;
    }
    //将0xC0000000的前4MB映射到物理地址，即虚拟地址0xC0000000-0xC0400000映射到物理地址0x00000000-0x00400000
    //VGA设备会用到，且内存探测时存储了数据在0x8000处，避免后面出现缺页
    for(unsigned int i=0,j=0;i<page_table_size;i++,j+=vmm_page_size){
        pt[i]=j|vmm_page_present|vmm_page_rw|vmm_page_kernel;
    }
    //将内核开始的前4MB映射到物理地址，即虚拟地址0xC1000000-0xC1400000映射到物理地址0x01000000-0x01400000
    for(unsigned int i=0,j=0x1000000;i<page_table_size;i++,j+=vmm_page_size){
        pt1[i]=j|vmm_page_present|vmm_page_rw|vmm_page_kernel;
    }
    //将内核开始的4MB-8MB映射到物理地址，即虚拟地址0xC1400000-0xC1800000映射到物理地址0x01400000-0x01800000
    for(unsigned int i=0,j=0x1000000+vmm_page_size*page_table_size;i<page_table_size;i++,j+=vmm_page_size){
        pt2[i]=j|vmm_page_present|vmm_page_rw|vmm_page_kernel;
    }
    //将内核开始的8MB-12MB映射到物理地址，即虚拟地址0xC1800000-0xC1B00000映射到物理地址0x01800000-0x01B00000
    for(unsigned int i=0,j=0x1000000+vmm_page_size*page_table_size*2;i<page_table_size;i++,j+=vmm_page_size){
        pt3[i]=j|vmm_page_present|vmm_page_rw|vmm_page_kernel;
    }
    //计算栈底在页表中的第几项，栈是向低地址增长的，实际栈为0xF7FFE000-0xF8000000，映射到0x37FFE000-0x38000000
    unsigned int stack_pt_idx=(((KERNEL_STACK_START-KERNEL_STACK_SIZE)&page_mask)/vmm_page_size)&0x3ff;
    for(unsigned int i=stack_pt_idx,j=0x37FFE000;i<stack_pt_idx+2;i++,j+=vmm_page_size){
        stack_pt[i]=j|vmm_page_present|vmm_page_rw|vmm_page_kernel;
    }
    /*unsigned int user_pt_idx=(((unsigned int)0x800000&page_mask)/vmm_page_size)&0x3ff;
    for(unsigned int i=user_pt_idx,j=(unsigned int)0x800000;i<stack_pt_idx+1;i++,j+=vmm_page_size){
        user_pt[i]=j|vmm_page_present|vmm_page_rw|vmm_page_kernel;
    }*/
    //0x37FFE000
    /*  开启分页    */
	__asm__ volatile ("mov %0, %%cr3" : : "r" (pdt) );       
	unsigned int cr0;
	__asm__ volatile ("mov %%cr0, %0" : "=r" (cr0) );
	// 最高位 PG 位置 1，分页开启
	cr0 |= (1u << 31);      
	__asm__ volatile ("mov %0, %%cr0" : : "r" (cr0) );

    /*设置栈顶为0xF8000000 ，栈大小为8KB*/
    __asm__ volatile ("mov %0, %%esp" : : "r" ((unsigned int)KERNEL_STACK_START));
	__asm__ volatile ("xor %%ebp, %%ebp" : :);
//__asm__ volatile ("mov %0, %%ebp" : : "r" ((unsigned int)KERNEL_STACK_START-(unsigned int)KERNEL_STACK_SIZE));
    //判断内核是否映射完全，关键是BSS段
    //ASSERT((unsigned int)(&kernel_end)>(unsigned int)0x01B00000);
    //调用内核入口
    main();
    
    return;
}
```

#### Init.h

```
#ifndef _INIT_H_
#define _INIT_H_

#include "../main/main.h"

#define vmm_page_size 0x1000
#define page_table_size 0x400
#define page_dir_size 0x400

#define page_mask 0xFFFFF000

#define vmm_page_present 0x1 
#define vmm_page_rw      0x2
#define vmm_page_kernel  0x0

/*
*__attribute__( (section(".init.data") ) )将其设置为特定的.init.data节
*方便在链接脚本中区分init部分和kernel部分
*/
__attribute__( (section(".init.data") ) ) unsigned int pdt[page_dir_size]__attribute__( (aligned(vmm_page_size) ) );
__attribute__( (section(".init.data") ) ) unsigned int pt_init[page_table_size]__attribute__( (aligned(vmm_page_size) ) );
//专门为VGA设备映射建立的页表
__attribute__( (section(".init.data") ) ) unsigned int pt[page_table_size]__attribute__( (aligned(vmm_page_size) ) );
//目前内核还不大，假定其大小不超过12MB，即三个页表
__attribute__( (section(".init.data") ) ) unsigned int pt1[page_table_size]__attribute__( (aligned(vmm_page_size) ) );
__attribute__( (section(".init.data") ) ) unsigned int pt2[page_table_size]__attribute__( (aligned(vmm_page_size) ) );
__attribute__( (section(".init.data") ) ) unsigned int pt3[page_table_size]__attribute__( (aligned(vmm_page_size) ) );
//__attribute__( (section(".init.data") ) ) unsigned int pt4[page_table_size]__attribute__( (aligned(vmm_page_size) ) );
//内核栈大小为8KB,起始地址为0xF8000000，只需一个页表即可
__attribute__( (section(".init.data") ) ) unsigned int stack_pt[page_table_size]__attribute__( (aligned(vmm_page_size) ) );
//user部分
__attribute__( (section(".init.data") ) ) unsigned int user_pt[page_table_size]__attribute__( (aligned(vmm_page_size) ) );


__attribute__( (section(".init.text") ) ) void init();

extern void main(void);

#endif
```

该文件是为了内核代码前开启分页，主要解决下面的问题：

在ld文件中，我们必须确定内核代码的VMA(虚拟空间地址)和LMA(物理空间地址)，如果在这之前不开启分页，那么内核代码的虚拟地址和物理地址均是从16MB处开始，若在这之后开启分页，那么整个虚拟地址空间就很难改变了（链接确定虚拟地址），所以会出错。

现在的解决方案时，使用init.c建立一个临时页表，并开启分页，然后把init代码放在1MB处（VMA和LMA均为1MB），而内核代码则放在VMA=0xC1000000处，LMA=0x01000000(16MB)处，因为不开启分页，那么VMA然后init代码执行完毕，跳转到内核代码后，这部分就没用了，所以直接新建一个页表，然后就可以删除这部分代码。

当然还有一种方案，就是qemu模拟器申请4GB内存，然后把VMA和LMA都放在0xC1000000处，在建立页表后，就把内核移到0x01000000处，这种方法笔者也试过，但是当用模拟器申请4G内存时，3.5G以上是不可用的，所以这种方案在使用模拟器时不可取，当使用真机时可能有效，读者们可以自行尝试。

init部分主要功能就是建立临时页表，主要包括内核代码部分和内核栈部分的映射，从下图的elf文件解析中我们可以看出需要3个页目录项，也就是12MB，才能包含bss段。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-LD2ECGGA-1609672391911)(readme.assets/image-20210102105209154.png)\]](https://img-blog.csdnimg.cn/20210103191410490.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


然后映射下内核栈，接着就把页目录表地址读入cr3寄存器，需要注意的是，cr3寄存器存储的都是物理地址，但由于init部分在链接脚本中没有指定虚拟地址，所以其虚拟地址=物理地址，接着开启cr0寄存器的PG位，从而打开分页模式，然后将esp指针指向内核栈的栈顶，最后跳转到内核的main函数，自此，init部分结束。
