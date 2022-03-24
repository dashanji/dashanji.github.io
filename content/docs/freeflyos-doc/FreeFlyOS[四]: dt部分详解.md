---
title: Chapter-4 dt部分详解
description: Chapter 4 of FreeFlyOS
toc: true
authors:
tags:
weight: 4
categories:
series:
date: '2022-03-24'
lastmod: '2022-03-24'
draft: false
---
## dt.c

主要功能：GDT和IDT表的建立和加载

1、GDT表在bootsector.S中就建立了，但那处于16位实模式下，而此时程序已进入32位保护模式，GDT每个段描述符限长变成4GB，所以需要建立一个新的GDT表，并加载到GDTR寄存器中。

2、IDT表和GDT表不同的是需要设置中断号和中断向量，并将硬件PIC上的中断号和中断向量绑定，当发生一个硬件中断时，比如按下键盘，PIC就会向系统送一个中断号（硬件完成），然后就会去IDT表中找该中断号对应的中断向量，然后执行相应的中断服务处理程序。

```
#include "dt.h"
#include "../asm/asm.h"
#include "../mem/memlayout.h"
/* *
 * Global Descriptor Table:
 *
 * The kernel and user segments are identical (except for the DPL). To load
 * the %ss register, the CPL must equal the DPL. Thus, we must duplicate the
 * segments for the user and the kernel. Defined as follows:
 *   - 0x0 :  unused (always faults -- for trapping NULL far pointers)
 *   - 0x8 :  kernel code segment
 *   - 0x10:  kernel data segment
 *   - 0x18:  user code segment
 *   - 0x20:  user data segment
 *   - 0x28:  defined for tss
 * */
struct segdesc gdt[] = {
    SEG_NULL, //null
    SEG(STA_X | STA_R, 0x0, 0xFFFFFFFF, DPL_KERNEL),  //kernel text
    SEG(STA_W, 0x0, 0xFFFFFFFF, DPL_KERNEL),   //kernel data
    SEG(STA_X | STA_R, 0x0, 0xFFFFFFFF, DPL_USER),  //user text
    SEG(STA_W, 0x0, 0xFFFFFFFF, DPL_USER),   //user data
    SEG_NULL,  //tss
    SEG_NULL   //ldt0
};

/* *
 * Interrupt descriptor table:
 *
 * Must be built at run time because shifted function addresses can't
 * be represented in relocation records.
 * */
static struct gatedesc idt[256] = {{0}};

/* *
 * set gdt's info
 * */
static struct dtdesc gdtinfo={
    sizeof(gdt)-1,(unsigned int)gdt
};

/* *
 * set tss
 * */
static struct taskstate ts = {0};

/* *
 * set idt's info
 * */
static struct dtdesc idtinfo = {
    sizeof(idt) - 1, (unsigned int)idt
};

/* *
 * lgdt - load the global descriptor table register and reset the
 * data/code segement registers for kernel.
 * */
static inline void lgdt(struct dtdesc *dt){
    asm volatile ("lgdt (%0)" :: "r" (dt));
    asm volatile ("movw %%ax, %%gs" :: "a" (USER_DS));
    asm volatile ("movw %%ax, %%fs" :: "a" (USER_DS));
    asm volatile ("movw %%ax, %%es" :: "a" (KERNEL_DS));
    asm volatile ("movw %%ax, %%ds" :: "a" (KERNEL_DS));
    asm volatile ("movw %%ax, %%ss" :: "a" (KERNEL_DS));
    // reload cs
    asm volatile ("ljmp %0, $1f\n 1:\n" :: "i" (KERNEL_CS));
}

/* *
 * 加载任务寄存器
 * */
void ltr(unsigned short sel) {
    asm volatile ("ltr %0" :: "r" (sel) : "memory");
}

/* *
 * 加载全局描述符表 
 * */
void gdt_init(){
    // set boot kernel stack and default SS0
    ts.ts_esp0=(unsigned int)KERNEL_STACK_START;
    ts.ts_ss0 = KERNEL_DS;

    // initialize the TSS filed of the gdt
    gdt[SEG_TSS] = SEGTSS(STS_T32A, (unsigned int)&ts, sizeof(ts), DPL_KERNEL);

    // reload all segment registers
    lgdt(&gdtinfo);

    // load the TSS
    ltr(GD_TSS);
}

/* *
 * 加载中断描述符表 
 * */
static inline void lidt(struct dtdesc *dt) {
    asm volatile ("lidt (%0)" :: "r" (dt) : "memory");
}

/* *
 *  
 *  将中断向量号和中断向量进行绑定    
 *        加载中断描述符表 
 * 
 * */
void idt_init(){
    extern unsigned int __vectors[];
    int i;
    for (i = 0; i < sizeof(idt) / sizeof(struct gatedesc); i ++) {
        SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
    }
    SETGATE(idt[T_SYSCALL], 1, GD_KTEXT, __vectors[T_SYSCALL], DPL_USER);
    //测试专用
    SETGATE(idt[0x60], 1, GD_KTEXT, __vectors[0x60], DPL_USER);
    lidt(&idtinfo);
}

/* *
 * 设置TSS的内核栈(ring0权限)
 * */
void set_ts_esp0(unsigned int esp){
    ts.ts_esp0=esp;
}
```



## dt.h

gdt和idt的属性设置

```
#ifndef _DT_H_
#define _DT_H_

/* Application segment type bits */
#define STA_X           0x8         // Executable segment
#define STA_E           0x4         // Expand down (non-executable segments)
#define STA_C           0x4         // Conforming code segment (executable only)
#define STA_W           0x2         // Writeable (non-executable segments)
#define STA_R           0x2         // Readable (executable segments)
#define STA_A           0x1         // Accessed

/* global descrptor numbers */
#define GD_KTEXT    ((1) << 3)      // kernel text
#define GD_KDATA    ((2) << 3)      // kernel data
#define GD_UTEXT    ((3) << 3)      // user text
#define GD_UDATA    ((4) << 3)      // user data
#define GD_TSS      ((5) << 3)        // task segment selector
#define GD_TSS1     ((6) << 3)        // task1 segment selector
#define DPL_KERNEL  (0)
#define DPL_USER    (3)

//global descriptor selector
#define KERNEL_CS   (GD_KTEXT|DPL_KERNEL)
#define KERNEL_DS   (GD_KDATA|DPL_KERNEL)
#define USER_CS     (GD_UTEXT|DPL_USER)
#define USER_DS     (GD_UDATA|DPL_USER)

/* System segment type bits */
#define STS_CG32        0xC         // 32-bit Call Gate
#define STS_IG32        0xE         // 32-bit Interrupt Gate
#define STS_TG32        0xF         // 32-bit Trap Gate

#define STS_T32A        0x9         // Available 32-bit TSS
#define SEG_TSS         0x5
#define SEG_TSS1        0x6

#define SEG_NULL                                            \
    (struct segdesc) {0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0}

#define SEG(type, base, lim, dpl)                           \
    (struct segdesc) {                                      \
        ((lim) >> 12) & 0xffff, (base) & 0xffff,            \
        ((base) >> 16) & 0xff, type, 1, dpl, 1,             \
        (unsigned)(lim) >> 28, 0, 0, 1, 1,                  \
        (unsigned) (base) >> 24                             \
    }

#define SEGTSS(type, base, lim, dpl)                        \
    (struct segdesc) {                                      \
        (lim) & 0xffff, (base) & 0xffff,                    \
        ((base) >> 16) & 0xff, type, 0, dpl, 1,             \
        (unsigned) (lim) >> 16, 0, 0, 1, 0,                 \
        (unsigned) (base) >> 24                             \
    }

#define T_SYSCALL           0x80

/* *
 * Set up a normal interrupt/trap gate descriptor
 *   - istrap: 1 for a trap (= exception) gate, 0 for an interrupt gate
 *   - sel: Code segment selector for interrupt/trap handler
 *   - off: Offset in code segment for interrupt/trap handler
 *   - dpl: Descriptor Privilege Level - the privilege level required
 *          for software to invoke this interrupt/trap gate explicitly
 *          using an int instruction.
 * */
#define SETGATE(gate, istrap, sel, off, dpl) {               \
        (gate).gd_off_15_0 = (unsigned int)(off) & 0xffff;      \
        (gate).gd_ss = (sel);                                \
        (gate).gd_args = 0;                                 \
        (gate).gd_rsv1 = 0;                                 \
        (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
        (gate).gd_s = 0;                                    \
        (gate).gd_dpl = (dpl);                              \
        (gate).gd_p = 1;                                    \
        (gate).gd_off_31_16 = (unsigned int)(off) >> 16;        \
    }

/* segment descriptors : bit field */
struct segdesc{
    unsigned sd_lim_15_0 : 16;      // low bits of segment limit
    unsigned sd_base_15_0 : 16;     // low bits of segment base address
    unsigned sd_base_23_16 : 8;     // middle bits of segment base address
    unsigned sd_type : 4;           // segment type (see STS_ constants)
    unsigned sd_s : 1;              // 0 = system, 1 = application
    unsigned sd_dpl : 2;            // descriptor Privilege Level
    unsigned sd_p : 1;              // present
    unsigned sd_lim_19_16 : 4;      // high bits of segment limit
    unsigned sd_avl : 1;            // unused (available for software use)
    unsigned sd_rsv1 : 1;           // reserved
    unsigned sd_db : 1;             // 0 = 16-bit segment, 1 = 32-bit segment
    unsigned sd_g : 1;              // granularity: limit scaled by 4K when set
    unsigned sd_base_31_24 : 8;     // high bits of segment base address
};

/* Gate descriptors for interrupts and traps */
struct gatedesc {
    unsigned gd_off_15_0 : 16;      // low 16 bits of offset in segment
    unsigned gd_ss : 16;            // segment selector
    unsigned gd_args : 5;           // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;           // reserved(should be zero I guess)
    unsigned gd_type : 4;           // type(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;              // must be 0 (system)
    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
    unsigned gd_p : 1;              // Present
    unsigned gd_off_31_16 : 16;     // high bits of offset in segment
};

//describe the gdt/ldt/idt，remember struct's member is first member in low address 
struct dtdesc{
    unsigned short dt_size;
    unsigned int dt_base;
}__attribute__((packed));

/* task state segment format (as described by the Pentium architecture book) */
struct taskstate {
    unsigned int ts_link;       // old ts selector
    unsigned int ts_esp0;      // stack pointers and segment selectors
    unsigned short ts_ss0;        // after an increase in privilege level
    unsigned short ts_padding1;  //Reserved bits.set to 0
    unsigned int ts_esp1;
    unsigned short ts_ss1;
    unsigned short ts_padding2;  //Reserved bits.set to 0
    unsigned int ts_esp2;
    unsigned short ts_ss2;
    unsigned short ts_padding3;   //Reserved bits.set to 0
    unsigned int ts_cr3;       // page directory base
    unsigned int ts_eip;       // saved state from last task switch
    unsigned int ts_eflags;
    unsigned int ts_eax;        // more saved state (registers)
    unsigned int ts_ecx;
    unsigned int ts_edx;
    unsigned int ts_ebx;
    unsigned int ts_esp;
    unsigned int ts_ebp;
    unsigned int ts_esi;
    unsigned int ts_edi;
    unsigned short ts_es;         // even more saved state (segment selectors)
    unsigned short ts_padding4;  //Reserved bits.set to 0
    unsigned short ts_cs;
    unsigned short ts_padding5;  //Reserved bits.set to 0
    unsigned short ts_ss;
    unsigned short ts_padding6;  //Reserved bits.set to 0
    unsigned short ts_ds;
    unsigned short ts_padding7;  //Reserved bits.set to 0
    unsigned short ts_fs;
    unsigned short ts_padding8;  //Reserved bits.set to 0
    unsigned short ts_gs;
    unsigned short ts_padding9;  //Reserved bits.set to 0
    unsigned short ts_ldt;
    unsigned short ts_padding10;  //Reserved bits.set to 0
    unsigned short ts_t;          // trap on task switch
    unsigned short ts_iomb;       // i/o map base address
} __attribute__((packed));

static inline void lgdt(struct dtdesc *dt);
static inline void lidt(struct dtdesc *dt);
void ltr(unsigned short sel);
void lidt(struct dtdesc *dt);
void gdt_init();
void idt_init();

void set_ts_esp0(unsigned int esp);
#endif
```

大概说下我理解的gdt和idt吧，详情可见intel手册或者博客。

gdt一般分为3个部分，一个部分为内核使用的段，KERNEL_CS也就是kernel code segment（内核代码段），KERNEL_DS也就是kernel data segment（内核数据段），一个部分是用户使用的段，USER_CS也就是user code segment（用户代码段），USER_DS是user data segment（用户数据段），最后一部分为TSS段，为什么要分为这3部分呢，就我的理解，内核段和用户段主要是为了区分内核代码和用户代码，从而保护内核数据，现代的TSS段一般没多大作用，因为以前的TSS段可以起到分割进程的作用，比如每个任务占有多少MB，多少个任务就有多少个TSS，但现在由于使用虚拟地址，每个任务都看似独占4G空间，以前由TSS界定进程地址空间，现在由进程自己的页表指定，而现在TSS的作用一般是用户代码切换到内核代码时，会自动切换到TSS设置的内核栈。

Idt主要是通过中断门实现的，如果大概理解下是这样的，这个中断门是我们人为设置的，同时还需绑定中断向量，当输入int 0x20时，表示我们访问0x20号的中断门，首先产生一个中断，硬件自动压栈，此后CS和SS会变为内核段代码，esp为我们设置的内核栈，eip为我们设置的中断向量，这均由硬件自动完成。如下图所示。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-EsXtfBuX-1609678119800)(readme.assets/image-20210101123853989.png)\]](https://img-blog.csdnimg.cn/20210103204911852.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


由于已经绑定好中断向量，接着就会执行对应的中断向量，如下图所示。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-YhSHRRve-1609678119801)(readme.assets/image-20210101122550051.png)\]](https://img-blog.csdnimg.cn/20210103204930620.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


其实中断向量就是一个地址，然后会调用相应地址的程序，如下所示。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-XBVkV7m8-1609678119802)(readme.assets/image-20210101122704418.png)\]](https://img-blog.csdnimg.cn/20210103204950755.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-a08nT3q4-1609678119803)(readme.assets/image-20210101130142268.png)\]](https://img-blog.csdnimg.cn/20210103205007699.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


接着就会保存当前的段寄存器，然后ds和es段寄存器切换到内核的数据段，当执行完中断程序就会切回原来特权级下的代码和数据。

再说下为什么设置一个trapret吧，是这样的，当我们需要一个用户进程的时候，怎么实现从KERNEL_CS段到USER_CS段的跨越呢，中断是一个很好的方法，我们在中断栈中保存用户段的状态，并设置进程的EIP为__trapret，那么这就模拟了一个中断处理程序完成后返回的操作，把中断栈的数据放出来后，自然就变成USER_CS段了，这就实现了用户进程的初始化，即CS为USER_CS。

另外需要注意的是系统调用门需要设置DPL为user，这样用户就可以进行系统调用了，默认中断向量号为0x80.如下图。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-esnUIF3D-1609678119804)(readme.assets/image-20210101122530922.png)\]](https://img-blog.csdnimg.cn/20210103205025876.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


关于intel中断详情可见

https://blog.csdn.net/huang987246510/article/details/88954782?utm_medium=distribute.pc_relevant.none-task-blog-baidujs_title-7&spm=1001.2101.3001.4242
