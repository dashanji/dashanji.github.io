---
title: Chapter-6 interrupt部分详解
description: Chapter 6 of FreeFlyOS
toc: true
authors:
tags:
weight: 6
categories:
series:
date: '2022-03-24'
lastmod: '2022-03-24'
draft: false
---
## trap.c

中断服务程序IRQ的实现.

```
#include "trap.h"
#include "../asm/asm.h"
#include "../vga/vga.h"
#include "../timer/timer.h"
#include "../debug/debug.h"
#include "../dt/dt.h"
#include "../task/task.h"
#include "../sync/sync.h"
#include "../serial/serial.h"
extern struct semaphore user_sema;

extern unsigned int volatile jiffies; //记录当前系统开机的时钟节拍数
extern unsigned int volatile second; //记录秒数
//int test_test=0;
extern struct task_struct *current;  //指向当前进程
static const char *IA32flags[] = {
    "CF", NULL, "PF", NULL, "AF", NULL, "ZF", "SF",
    "TF", "IF", "DF", "OF", NULL, NULL, "NT", NULL,
    "RF", "VM", "AC", "VIF", "VIP", "ID", NULL, NULL,
};
void print_regs(struct pushregs *regs) {
    printk("  edi  0x%08ux\n", regs->reg_edi);
    printk("  esi  0x%08ux\n", regs->reg_esi);
    printk("  ebp  0x%08ux\n", regs->reg_ebp);
    printk("  oesp 0x%08ux\n", regs->reg_oesp);
    printk("  ebx  0x%08ux\n", regs->reg_ebx);
    printk("  edx  0x%08ux\n", regs->reg_edx);
    printk("  ecx  0x%08ux\n", regs->reg_ecx);
    printk("  eax  0x%08ux\n", regs->reg_eax);
}
void print_trapframe(struct trapframe *tf) {
   // printk("trapframe at %p\n", tf);
    print_regs(&tf->tf_regs);
    printk("  ds   0x----%04ux\n", tf->tf_ds);
    printk("  es   0x----%04ux\n", tf->tf_es);
    printk("  fs   0x----%04ux\n", tf->tf_fs);
    printk("  gs   0x----%04ux\n", tf->tf_gs);
    //printk("  trap 0x%08ux %s\n", tf->tf_trapno, trapname(tf->tf_trapno));
    printk("  err  0x%08ux\n", tf->tf_err);
    printk("  eip  0x%08ux\n", tf->tf_eip);
    printk("  cs   0x----%04ux\n", tf->tf_cs);
    printk("  flag 0x%08ux ", tf->tf_eflags);

    int i, j;
    for (i = 0, j = 1; i < sizeof(IA32flags) / sizeof(IA32flags[0]); i ++, j <<= 1) {
        if ((tf->tf_eflags & j) && IA32flags[i] != NULL) {
            printk("%s,", IA32flags[i]);
        }
    }
    printk("IOPL=%d\n", (tf->tf_eflags & FL_IOPL_MASK) >> 12);
    while(1);
   // if (!trap_in_kernel(tf)) {
   //     printk("  esp  0x%08x\n", tf->tf_esp);
   //     printk("  ss   0x----%04x\n", tf->tf_ss);
   // }
}

/* *
 * trap - handles or dispatches an exception/interrupt. if and when trap() returns,
 * the code in kern/trap/trapentry.S restores the old CPU state saved in the
 * trapframe and then uses the iret instruction to return from the exception.
 * */
/* trap_dispatch - dispatch based on what type of trap occurred */
static void trap_dispatch(struct trapframe *tf) 
{
    char c;
    switch (tf->tf_trapno) {
        case IRQ_TEST:
            printk("test user trap\n");
            break;
        case T_PGFLT:
            print_trapframe(tf);
            //printk("queye\n");
            break;
        case T_SYSCALL:
            syscall_trap(tf);
            break;
        case IRQ_OFFSET + IRQ_TIMER:
            jiffies++;
            //1秒触发一次
            if (jiffies % 100 == 0){
                current->counter--;
                second++;
                //printk("current->counter:%08d\n",current->counter);
            }
            if(current->counter==0){
                //printk("Start Schedule\n",current->counter);
                schedule();
            }
            break;
        case IRQ_OFFSET + IRQ_COM1:
             c = cons_getc();
             //cons_putc(c);
             break;
        case IRQ_OFFSET + IRQ_KBD:
            c = cons_getc();
            //printk("%c",c);
            //cons_putc(c);
            //printk("anle\n");
            //test_test++;
            //printk("test_test:%02d\n",test_test);
            //printk("user_sema.value:%08d\n",user_sema.value);
            if(user_sema.value==0){
                 sema_up(&user_sema);
                 schedule();
                //printk("user_sema.value:%08d\n",user_sema.value);
            }
               
            //printk("rpos:%08x",cons.rpos);
            //printk("wpos:%08x",cons.wpos);
            //printk("cons.buf:%c",cons.buf[cons.wpos-1]);
            break;
        case IRQ_OFFSET+IRQ_IDE1:
           /* struct ide_channel* channel = &channels[0];
            if (channel->expecting_intr) {
                channel->expecting_intr = 0;
                sema_up(&channel->disk_done);
                // 读取状态寄存器使硬盘控制器认为此次的中断已被处理,从而硬盘可以继续执行新的读写 
                inb(reg_status(channel));
            } */
            break;
        case IRQ_OFFSET+IRQ_IDE2:
            /*struct ide_channel* channel = &channels[1];
            if (channel->expecting_intr) {
                channel->expecting_intr = 0;
                sema_up(&channel->disk_done);
                 读取状态寄存器使硬盘控制器认为此次的中断已被处理,从而硬盘可以继续执行新的读写 
                inb(reg_status(channel));
            }  */
            break;
        default:
            // in kernel, it must be a mistake
            printk("unexpected trap in kernel!\n");
    }
}

void trap(struct trapframe *tf) {
    trap_dispatch(tf);
}

void disable_interupt(){
    // 关闭中断
    asm volatile ("cli");
}

void enable_interupt(){
    // 开启中断
    asm volatile ("sti");
}
/*
**   获取当前中断状态
*/
enum intr_status get_now_intr_status(){
    unsigned int eflags=0;
    get_intr_status(eflags);
    return (EFLAGS_IF & eflags) ? INTR_ON : INTR_OFF ;
}
/*
**   开中断并获取之前的中断状态
*/
enum intr_status intr_enable(){
    if(get_now_intr_status()==INTR_OFF){
        enable_interupt();
        return INTR_OFF;
    }
    else{
        return INTR_ON;
    }
}
/*
**   关中断并获取之前的中断状态
*/
enum intr_status intr_disable(){
    if(get_now_intr_status()==INTR_ON){
        disable_interupt();
        return INTR_ON;
    }
    else{
        return INTR_OFF;
    }
}

/*
**      临界区访问,保存中断状态后
**           <关闭中断>
*/
enum intr_status intr_save(){
    enum intr_status status;
    status=intr_disable();
    return status;
}
/*
**      临界区访问,保存中断状态后
**           <关闭中断>
*/
void intr_restore(enum intr_status status){
    if(status==INTR_ON)
        enable_interupt();
}
```



## trap.h

trap号，硬件IRQ号，两者主要区别是陷阱门发生后不会关中断，可能会出现中断嵌套，而中断门发生后会关闭中断，防止出现中断嵌套。

```
#ifndef _TRAP_H_
#define _TRAP_H_

/* Trap Numbers */
/* Processor-defined: */
#define T_DIVIDE                0   // divide error
#define T_DEBUG                 1   // debug exception
#define T_NMI                   2   // non-maskable interrupt
#define T_BRKPT                 3   // breakpoint
#define T_OFLOW                 4   // overflow
#define T_BOUND                 5   // bounds check
#define T_ILLOP                 6   // illegal opcode
#define T_DEVICE                7   // device not available
#define T_DBLFLT                8   // double fault
// #define T_COPROC             9   // reserved (not used since 486)
#define T_TSS                   10  // invalid task switch segment
#define T_SEGNP                 11  // segment not present
#define T_STACK                 12  // stack exception
#define T_GPFLT                 13  // general protection fault
#define T_PGFLT                 14  // page fault
// #define T_RES                15  // reserved
#define T_FPERR                 16  // floating point error
#define T_ALIGN                 17  // aligment check
#define T_MCHK                  18  // machine check
#define T_SIMDERR               19  // SIMD floating point error

/* Hardware IRQ numbers. We receive these as (IRQ_OFFSET + IRQ_xx) */
#define IRQ_OFFSET              32  // IRQ 0 corresponds to int IRQ_OFFSET

#define IRQ_TIMER               0
#define IRQ_KBD                 1
#define IRQ_COM1                4
#define IRQ_IDE1                14
#define IRQ_IDE2                15
#define IRQ_ERROR               19
#define IRQ_SPURIOUS            31

#define IRQ_TEST  0x60
/*
 * These are arbitrarily chosen, but with care not to overlap
 * processor defined exceptions or interrupt vectors.
 * */
#define T_SWITCH_TOU                120    // user/kernel switch
#define T_SWITCH_TOK                121    // user/kernel switch

#define NULL ((void *)0)
#define FL_IOPL_MASK    0x00003000  // I/O Privilege Level bitmask

/*
**   中断状态
*/
enum intr_status{
    INTR_OFF=0,
    INTR_ON=1,
};
#define EFLAGS_IF 0x00000200  //中断标志位
#define get_intr_status(eflag_val) asm volatile("pushfl ; popl %0":"=g"(eflag_val))

#define local_intr_save(x)      do { x = intr_save(); } while (0)
#define local_intr_restore(x)       intr_restore(x)

/* registers as pushed by pushal */
struct pushregs {
    unsigned int reg_edi;
    unsigned int reg_esi;
    unsigned int reg_ebp;
    unsigned int reg_oesp;          /* Useless */
    unsigned int reg_ebx;
    unsigned int reg_edx;
    unsigned int reg_ecx;
    unsigned int reg_eax;
}__attribute__((packed));

struct trapframe {
    struct pushregs tf_regs;
    unsigned short tf_gs;
    unsigned short tf_padding0;
    unsigned short tf_fs;
    unsigned short tf_padding1;
    unsigned short tf_es;
    unsigned short tf_padding2;
    unsigned short tf_ds;
    unsigned short tf_padding3;
    unsigned int tf_trapno;
    /* below here defined by x86 hardware */
    unsigned int tf_err;
    unsigned int tf_eip;
    unsigned short tf_cs;
    unsigned short tf_padding4;
    unsigned int tf_eflags;
    /* below here only when crossing rings, such as from user to kernel */
    unsigned int tf_esp;
    unsigned short tf_ss;
    unsigned short tf_padding5;
} __attribute__((packed));

void print_trapframe(struct trapframe *tf);
static void trap_dispatch(struct trapframe *tf);
void trap(struct trapframe *tf);
void disable_interupt();
void enable_interupt();
enum intr_status get_now_intr_status();
enum intr_status intr_enable();
enum intr_status intr_disable();
enum intr_status intr_save();
void intr_restore(enum intr_status status);

#endif
```



## trapentry.S

用于保存中断处理程序前的现场（上下文），当程序执行完毕后返回原来的现场，各种段寄存器恢复成原值。

```
#define GD_KDATA    ((2) << 3)      // kernel data
.code32
# vectors.S sends all traps here.
.section .text
.globl __alltraps
__alltraps:
    # push registers to build a trap frame
    # therefore make the stack look like a struct trapframe
    pushl %ds
    pushl %es
    pushl %fs
    pushl %gs
    pushal

    # load GD_KDATA into %ds and %es to set up data segments for kernel
    movl $GD_KDATA, %eax
    movw %ax, %ds
    movw %ax, %es

    # push %esp to pass a pointer to the trapframe as an argument to trap()
    pushl %esp

    # call trap(tf), where tf=%esp
    call trap

    # pop the pushed stack pointer
    popl %esp

    # return falls through to trapret...
.globl __trapret
__trapret:
    # restore registers from stack
    popal

    # restore %ds, %es, %fs and %gs
    popl %gs
    popl %fs
    popl %es
    popl %ds

    # get rid of the trap number and error code
    addl $0x8, %esp
    iret

.globl forkrets
forkrets:
    # set stack to this new task's trapframe
    movl 4(%esp), %esp
    jmp __trapret
```



## vector.S

中断向量表，在IDT初始化时和中断号绑定，初始化了256个中断向量，实际只用了几个。

## syscall.c

```
#include "syscall.h"
#include "../task/task.h"
#include "../vga/vga.h"
#include "../file/fs.h"
#include "../mem/vmm.h"
#include "../task/exec.h"
#include "../pipe/pipe.h"
extern struct task_struct *current;  //指向当前进程

static int
syscall_exit(unsigned int arg[]) {
    int status = (int)arg[0];
    sys_exit(status);
    return 0;
}

static int
sys_fork(unsigned int arg[]) {
    struct trapframe *tf = current->tf;
    unsigned int stack = tf->tf_esp;
    return do_fork(0, stack, tf);
}

static int
syscall_wait(unsigned int arg[]) {
    int *status = (int *)arg[0];
    return sys_wait(status);
    //return do_wait(pid, store);
}

static int
sys_exec(unsigned int arg[]) {
    const char *path = (const char *)arg[0];
    const char **argv = (const char **)arg[1];
    return sys_execv(path,argv);
}

static int
sys_yield(unsigned int arg[]) {
    //return do_yield();
}

static int
sys_kill(unsigned int arg[]) {
    int pid = (int)arg[0];
    //return do_kill(pid);
}

static int
sys_getpid(unsigned int arg[]) {
    return current->pid;
}

static int
sys_print_char(unsigned int arg[]) {
    char c = (char)arg[0];
    print_char(c,default_background,default_foreground);
    return 0;
}
static int
sys_print_string(unsigned int arg[]) {
    const char *str = (const char *)arg[0];
    print_string(str,default_background,default_foreground);
    return 0;
}
static int
sys_print_num(unsigned int arg[]) {
    int num = (int)arg[0];
    unsigned char base=(unsigned char)arg[1];
    char len=(char)arg[2];
    int flag=(int)arg[3];
    print_num(num,default_background,default_foreground,base,len,flag);
    return 0;
}
static int
sys_backtrace(unsigned int arg[]) {
    backtrace();
    return 0;
}
static int
sys_pgdir(unsigned int arg[]) {
    //print_pgdir();
    return 0;
}
static int
sys_fdread(unsigned int arg[]) {
    int fd=arg[0];
    void *buf=(void *)arg[1];
    unsigned int count=(unsigned int)arg[2];
    return sys_read(fd,buf,count);
}
static int
syscall_open(unsigned int arg[]) {
    const char* pathname=(const char*)arg[0];
    unsigned char flags=(unsigned char)arg[1];
    return sys_open(pathname,flags);
}
static int
syscall_close(unsigned int arg[]) {
    int fd=(int)arg[0];
    return sys_close(fd);
}
static int
syscall_write(unsigned int arg[]) {
    int fd=(int)arg[0];
    const void* buf=(const void*)arg[1];
    unsigned int count=(unsigned int)arg[2];
    return sys_write(fd,buf,count);
}
static int
syscall_lseek(unsigned int arg[]) {
    int fd=(int)arg[0];
    int offset=(int)arg[1];
    unsigned char whence=(unsigned char)arg[2];
    return sys_lseek(fd,offset,whence);
}
static int
syscall_unlink(unsigned int arg[]) {
    const char* pathname=(const char*)arg[0];
    return sys_unlink(pathname);
}
static int
syscall_mkdir(unsigned int arg[]) {
    const char* pathname=(const char*)arg[0];
    return sys_mkdir(pathname);
}
static int
syscall_rmdir(unsigned int arg[]) {
    const char* pathname=(const char*)arg[0];
    return sys_rmdir(pathname);
}
static int
syscall_rewinddir(unsigned int arg[]) {
    struct dir* dir=(struct dir*)arg[0];
    sys_rewinddir(dir);
    return 0;
}
static char*
syscall_getcwd(unsigned int arg[]) {
    char* buf=(char* )arg[0];
    unsigned int size=(unsigned int)arg[1];
    return sys_getcwd(buf,size);
}
static int
syscall_chdir(unsigned int arg[]) {
    const char* path=(const char*)arg[0];
    return sys_chdir(path);
}
static int
syscall_stat(unsigned int arg[]) {
    const char* path=(const char*)arg[0];
    struct stat *buf=(struct stat *)arg[1];
    return sys_stat(path,buf);
}
static struct dir *
syscall_opendir(unsigned int arg[]) {
    const char* name=(const char*)arg[0];
    return sys_opendir(name);
}
static int
syscall_closedir(unsigned int arg[]) {
    struct dir* dir=(struct dir* )arg[0];
    return sys_closedir(dir);
}
static int
syscall_readdir(unsigned int arg[]) {
    struct dir* dir=(struct dir* )arg[0];
    return sys_readdir(dir);
}
static int 
syscall_print_task(unsigned int arg[]) {
    sys_print_task();
    return 0;
}
static unsigned int 
syscall_malloc(unsigned int arg[]){
    unsigned int bytes=(unsigned int)arg[0];
    return sys_malloc(bytes);
}
static int
syscall_free(unsigned int arg[]){
    unsigned int addr=(unsigned int)arg[0];
    unsigned int size=(unsigned int)arg[1];
    sys_free(addr,size);
    return 0;
}
static int
syscall_pipe(unsigned int arg[]){
    unsigned int *fd=(unsigned int)arg[0];
    sys_pipe(fd);
    return 0;
}
static int (*syscalls[])(unsigned int arg[]) = {
    [SYS_exit]              syscall_exit,
    [SYS_fork]              sys_fork,
    [SYS_wait]              syscall_wait,
    [SYS_exec]              sys_exec,
    [SYS_yield]             sys_yield,
    [SYS_kill]              sys_kill,
    [SYS_getpid]            sys_getpid,
    [SYS_fdread]            sys_fdread, 
    [SYS_pgdir]             sys_pgdir,
    [SYS_print_char]        sys_print_char,
    [SYS_print_string]      sys_print_string,
    [SYS_print_num]         sys_print_num,
    [SYS_backtrace]         sys_backtrace,
    [SYS_open]              syscall_open,
    [SYS_close]             syscall_close,
    [SYS_write]             syscall_write,
    [SYS_lseek]             syscall_lseek,
    [SYS_unlink]            syscall_unlink,
    [SYS_mkdir]             syscall_mkdir,
    [SYS_rmdir]             syscall_rmdir,
    [SYS_rewinddir]         syscall_rewinddir,
    [SYS_getcwd]            syscall_getcwd,
    [SYS_chdir]             syscall_chdir,
    [SYS_stat]              syscall_stat,
    [SYS_opendir]           syscall_opendir,
    [SYS_closedir]          syscall_closedir,
    [SYS_readdir]           syscall_readdir,
    [SYS_print_task]        syscall_print_task,
    [SYS_malloc]            syscall_malloc,
    [SYS_free]              syscall_free,
    [SYS_mmap]              sys_mmap,
    [SYS_pipe]              syscall_pipe,
};

#define NUM_SYSCALLS        ((sizeof(syscalls)) / (sizeof(syscalls[0])))

void
syscall_trap(struct trapframe *tf) {
    //struct trapframe *tf = current->tf;
    unsigned int arg[5];
    int num = tf->tf_regs.reg_eax;
    if (num >= 0 && num < NUM_SYSCALLS) {
        if (syscalls[num] != NULL) {
            arg[0] = tf->tf_regs.reg_edx;
            arg[1] = tf->tf_regs.reg_ecx;
            arg[2] = tf->tf_regs.reg_ebx;
            arg[3] = tf->tf_regs.reg_edi;
            arg[4] = tf->tf_regs.reg_esi;
            tf->tf_regs.reg_eax = syscalls[num](arg);
            return ;
        }
    }
    
    //print_trapframe(tf);
    //printk("undefined syscall %d, pid = %d, name = %s.\n",
    //        num, current->pid, current->name);
}

```



## syscall.h

```
#ifndef _SYSCALL_H_
#define _SYSCALL_H_
#include "trap.h"
/* syscall number */
#define SYS_exit            1
#define SYS_fork            2
#define SYS_wait            3
#define SYS_exec            4
#define SYS_clone           5
#define SYS_yield           10
#define SYS_sleep           11
#define SYS_kill            12
#define SYS_gettime         17
#define SYS_getpid          18
#define SYS_brk             19
#define SYS_mmap            20
#define SYS_munmap          21
#define SYS_shmem           22

#define SYS_fdread            24

#define SYS_pgdir           31

#define SYS_print_char      36
#define SYS_print_string    37
#define SYS_print_num       38
#define SYS_backtrace       39
#define SYS_open  40
#define SYS_close 41
#define SYS_write 42
#define SYS_lseek 43
#define SYS_unlink 44
#define SYS_mkdir 45
#define SYS_rmdir 46
#define SYS_rewinddir 47
#define SYS_getcwd 48
#define SYS_chdir 49
#define SYS_stat 50
#define SYS_opendir 51
#define SYS_closedir 52
#define SYS_readdir 53
#define SYS_print_task 54
#define SYS_malloc 55
#define SYS_free 56
#define SYS_mmap 57
#define SYS_pipe 58
void syscall_trap(struct trapframe *tf);
int user_sys_getpid(void);
void user_print_char(char c); 
void user_print_string(char *str);
void user_print_num(int num,unsigned char base,char len,int flag);
#endif
```

关于中断，网上有很多详细的资料，背景类知识就不再赘述，大概说一下FreeFlyOS的中断体系把，主要包含两种，一种是硬件中断，比如缺页异常、时钟中断、键盘中断、串口中断、硬盘中断等等，这些中断处理程序设计的比较简单。还有一种是软中断，我们通过中断门实现的一种中断，主要用于系统调用，也就是ring3权限的用户想要对系统资源进行访问时,ring0权限的OS提供给用户访问的一种接口。大概讲一下系统调用怎么传递参数吧，首先我们看用户视角下的系统调用。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-SuNlJR6L-1609678882123)(readme.assets/image-20210102111920939.png)\]](https://img-blog.csdnimg.cn/20210103210145826.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


一般而言，系统调用的中断号是0x80，这个大家应该都知道，那么参数应该如何传递呢，这里我们规定最多只能传递5个参数，而且每个参数需要放在指定的寄存器中，首先把参数数量放在eax中，然后第一个参数放在edx参数，依次类推。

同样，内核中的中断服务程序会根据栈帧接受到这些信息，从而完成参数的传递。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-Pe9eu5B4-1609678882124)(readme.assets/image-20210102112300909.png)\]](https://img-blog.csdnimg.cn/20210103210216857.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


现在还需说明一个问题，我们使用中断的时候，硬件会自动压栈，只包含SS、ESP、EFLAGS、CS、EIP、ERROR等信息，而这些寄存器信息并不包含在内，所以我们需要构建一个参数传递栈帧，如下图所示。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-ESnmerRX-1609678882125)(readme.assets/image-20210102112516846.png)\]](https://img-blog.csdnimg.cn/20210103210235507.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


硬件压栈的部分我们就不需要继续压栈了，只需要对栈帧其他寄存器进行压栈即可，构造过程如下，注意pushal是压入所有通用寄存器。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-MlKQX4qK-1609678882126)(readme.assets/image-20210102113658283.png)\]](https://img-blog.csdnimg.cn/20210103210250283.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


就说这么多吧，拜拜。
