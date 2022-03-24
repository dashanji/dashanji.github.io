---
title: Chapter-12 debug部分详解
description: Chapter 12 of FreeFlyOS
toc: true
authors:
tags:
weight: 12
categories:
series:
date: '2022-03-24'
lastmod: '2022-03-24'
draft: false
---
#### debug.c

包含两个函数，一个是打印寄存器信息，另一个是设置断言失败的结果

```
#include "debug.h"
#include "monitor.h"
#include "../vga/vga.h"
#include "../interrupt/trap.h"

/*
*   打印段寄存器信息
*/
void print_seg()
{
    unsigned short cs,ds,gs,es,fs,ss;
    asm volatile("movw %%cs, %0;"
                 "movw %%ds, %1;" 
                 "movw %%gs, %2;" 
                 "movw %%es, %3;"
                 "movw %%fs, %4;"
                 "movw %%ss, %5;": "=m"(cs),"=m"(ds),"=m"(gs),"=m"(es),"=m"(fs),"=m"(ss)); 
    printk("cs=%04x\n",cs);
    printk("ds=%04x\n",ds);
    printk("gs=%04x\n",gs);
    printk("es=%04x\n",es);
    printk("fs=%04x\n",fs);
    printk("ss=%04x\n",ss);
}

/* *
 * __panic - __panic is called on unresolvable fatal errors. it prints
 * "panic: 'message'", and then enters the kernel monitor.
 * */
void
__PANIC(const char *file, int line, const char *func, const char *condition) {

    // 关中断
    intr_disable();

    // 打印错误信息
    printk("kernel panic at %s:%d:\n    ", file, line);
    printk("In %s , the condition(%s) is wrong\n",func,condition);
    
    //printk("stack trackback:\n");
    //print_seg();
    
    while (1) {
        monitor();
    }
}

```



#### debug.h

设置断言

```
#ifndef _DEBUG_H_
#define _DEBUG_H_

#define PANIC(...)                                      \
    __PANIC(__FILE__ , __LINE__ , __func__ , __VA_ARGS__)

#define ASSERT(x)                                   \
    if (!(x)) {                                     \
            PANIC(#x);                              \ 
    }                                               \
    else{}


void print_seg();
void __attribute__((noreturn)) 
__PANIC(const char *file, int line, const char *func, const char *condition);

#endif
```

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-LaLUjJBu-1609680977189)(readme.assets/image-20201231155706920.png)\]](https://img-blog.csdnimg.cn/20210103213641987.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


什么是断言呢，简单来讲就是当程序执行到某一步时，它的状态一定是我预料的状态，也就是我设置的条件必须为真，如果不为真，那么内核就会退出（PANIC），然后运行PANIC函数。这里简单说明下PANIC的几个参数，__FILE__、__LINE__和__func__为C语言中预定义的宏，分别代表当前文件名、当前函数名、运行当前文件的行数，而__VA_ARGS__则是预处理器专门提供的一个标识符，只允许在具有可变参数的宏替换列表中出现，表示该参数至少有一个，但可以为空。我们这里设置的PANIC函数除了前3个宏，只有一个参数，就是断言的判断条件。

#### monitor.c

该文件主要定义了一个类似shell的对话机制，首先需要初始化指令数组，然后分配一个监视器不断读取用户输入的字符，然后使用解析器将用户的字符和预设的指令名称进行匹配，若匹配成功则运行相应指令。

```
#include "monitor.h"
#include "readline.h"
#include "../vga/vga.h"
#include "../interrupt/trap.h"
#include "../asm/asm.h"
extern unsigned int shift;

/* 
** 初始化指令数组 
*/
struct instr instr_list[]={
    {"hello","The instruction is to welcome you!",instr_hello},
    {"help","You can use the following instructions!",instr_help},
    {"game","Play a simple game---Guess number!",instr_game}
};

#define instr_num (sizeof(instr_list)/sizeof(struct instr))

/* 
** 监视器：用于监控用户输入字符 
*/
void monitor(){
    char *buf;
    shift=0;
    while(1){
        if((buf=(readline("K<")))!=NULL){
            run(buf);
        }
    }
}
/* 
** 解析器：用于解析用户输入字符 
*/
void run(char *buf){
    for(int i=0;i<instr_num;i++){
        if(!strcmp(buf,instr_list[i].name)){
            instr_list[i].func();
        }
    }
    
}

/*
** hello指令
*/
void instr_hello(){
    printk("%s\n",instr_list[0].desc);
    printk("Nice to meet you!I'm the writer of FreeFlyOS!\n");
}

/*
** help指令
*/
void instr_help(){
    printk("%s\n",instr_list[1].desc);
    printk("Instruction ---- Describition\n");
    for(int i=0;i<instr_num;i++){
        printk("%s --- %s\n",instr_list[i].name,instr_list[i].desc);
    }
}

/*
** game指令
*/
void instr_game(){
    char *buf;
    int answer=28;
    int num;
    printk("%s\n",instr_list[2].desc);
    while(1){
        num=0;
        if((buf=(readline("Now please input a number:")))!=NULL){
            while(*buf!='\0'){
                num=num*10+(*buf-'0');
                buf++;
            }
            if(num==answer){
                printk("Congratulations to you!Your answer is correct\n");
                break;
            }
            else if(num<answer){
                printk("Your answer is smaller\n");
            }
            else{
                printk("Your answer is bigger\n");
            }
        } 
    }
}
```



#### monitor.h

monitor.c对应的头文件，其中设置了一个函数指针，用于处理不同指令。

```
#ifndef _MONITOR_H_
#define _MONITOR_H_

//指令
struct instr{
    const char *name;  //指令名
    const char *desc;  //指令描述
    int (*func)(); //函数指针，指令处理函数
};

void monitor();
void run(char *buf);
void instr_hello();
void instr_help();
void instr_game();

#endif
```



#### readline.c

主要是对输入的字符进行处理，当输入普通字符时，将其记录在buf中，输入退格键，则buf数组清一位，当输入回车键或者换行符，表示指令输入完成，输出buf。

```
#include "readline.h"
#include "../vga/vga.h"
#define BUFSIZE 1024
#define NULL ((void *)0)
static char buf[BUFSIZE];

/* 
** getchar - reads a single non-zero character from stdin 
*/
int
getchar(void) {
    int c;
    while ((c = cons_getc()) == 0)
        /* do nothing */;
    return c;
}
/* *
 * readline - get a line from stdin
 * @prompt:     the string to be written to stdout
 *
 * The readline() function will write the input string @prompt to
 * stdout first. If the @prompt is NULL or the empty string,
 * no prompt is issued.
 *
 * This function will keep on reading characters and saving them to buffer
 * 'buf' until '\n' or '\r' is encountered.
 *
 * Note that, if the length of string that will be read is longer than
 * buffer size, the end of string will be discarded.
 *
 * The readline() function returns the text of the line read. If some errors
 * are happened, NULL is returned. The return value is a global variable,
 * thus it should be copied before it is used.
 * */
char *
readline(const char *prompt) {
    if (prompt != NULL) {
        printk("%s", prompt);
    }
    int i = 0, c;
    while (1) {
        c = getchar(); //cons_getc();
        if (c < 0) {
            return NULL;
        }
        else if (c >= ' ' && i < BUFSIZE - 1) {
            cons_putc(c);
            buf[i ++] = c;
        }
        else if (c == '\b' && i > 0) {
            cons_putc(c);
            i --;
        }
        else if (c == '\n' || c == '\r') {
            cons_putc(c);
            buf[i] = '\0';
            return buf;
        }
    }
}


```



#### readline.h

readline.c对应的头文件，相对比较简单，没啥可讲的。

```
#ifndef _READLINE_H_
#define _READLINE_H_

int getchar(void);
char *readline(const char *prompt);
#endif
```

讲真的，断言其实没必要实现，但实现之后对我们调试有很大的帮助，因为当你无法判断某个函数会不会受某些事件（比如进程切换）影响时，就可以加上一个断言，确认其他事件的状态，保证函数能够正常运行，或者函数出错，再去对断言条件进行分析。呃，由于FreeFlyOS调试好了，基本不会出现断言错误，即monitor不会运行。之前是做这个是因为还没实现shell机制，所以想着先搞个对话机制，支持3种命令。hello、help和game，嘻嘻，一个简单的猜数字游戏，无聊时做的。。。


